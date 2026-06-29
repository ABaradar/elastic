# Ansible Role: `elasticsearch_cluster`

An Ansible role that installs and configures a multi-node Elasticsearch cluster on Debian/Ubuntu VMs.
It supports two deployment topologies and two installation methods, handles block-device formatting
and mounting, computes JVM heap automatically, and renders all Elasticsearch configuration from
templates.

---

## Table of Contents

1. [Supported Topologies](#1-supported-topologies)
2. [Repository Structure](#2-repository-structure)
3. [How the Role Works — Task Stages](#3-how-the-role-works--task-stages)
4. [Variables Reference](#4-variables-reference)
5. [Topology A — Dedicated Masters + Data Nodes](#5-topology-a--dedicated-masters--data-nodes)
6. [Topology B — All-in-One Nodes](#6-topology-b--all-in-one-nodes)
7. [Installation Methods](#7-installation-methods)
8. [Disk Formatting and Mounting](#8-disk-formatting-and-mounting)
9. [JVM Heap Sizing](#9-jvm-heap-sizing)
10. [Elasticsearch Configuration Template](#10-elasticsearch-configuration-template)
11. [Node Types and `configure.yml`](#11-node-types-and-configureyml)
12. [Rolling Upgrades (Major Version)](#12-rolling-upgrades-major-version)
13. [Safety and Operational Cautions](#13-safety-and-operational-cautions)

---

## 1. Supported Topologies

The role supports two mutually independent cluster topologies. You pick one by defining the
appropriate inventory groups and running the matching plays in `site.yml`.

| Topology | Inventory groups | Node roles in Elasticsearch |
|---|---|---|
| **Dedicated** | `[es_master]` + `[es_data]` | Masters are master-only; data nodes are data-only |
| **All-in-one** | `[es_all_in_one]` | Every node carries both `master` and `data` roles |

Both topologies go through the same underlying task stages; what differs is the value of
`es_node_type` passed to the role and which groups exist in inventory.

---

## 2. Repository Structure

```
.
├── hosts.ini                          # Inventory — edit IPs here
├── site.yml                           # Top-level playbook — runs all stages in order
└── elasticsearch_cluster/             # The Ansible role
    ├── defaults/
    │   └── main.yml                   # All default variable values
    ├── files/
    │   └── *.deb                      # Place local .deb packages here (deb install method)
    ├── handlers/
    │   └── main.yml                   # Restarts elasticsearch on config change
    ├── tasks/
    │   ├── main.yml                   # Entry point — includes stages via feature flags
    │   ├── update_hosts.yml           # Writes /etc/hosts entries for all cluster nodes
    │   ├── checks.yml                 # Preflight assertions (inventory, OS, disk)
    │   ├── format_mount.yml           # Formats block device and mounts it
    │   ├── packages.yml               # Installs apt prerequisites + Elastic APT repo
    │   ├── install.yml                # Installs Elasticsearch (repo or local deb)
    │   ├── configure.yml              # Renders elasticsearch.yml and JVM options
    │   └── upgrade.yml                # Rolling per-node upgrade (disable alloc, stop, install, wait green)
    └── templates/
        ├── elasticsearch.yml.j2       # Elasticsearch node configuration
        └── jvm_heap.options.j2        # JVM heap and GC options
```

---

## 3. How the Role Works — Task Stages

`tasks/main.yml` is the entry point. It includes each stage file conditionally using feature-flag
variables. This design means you can run the role multiple times against the same hosts with
different sets of flags — each invocation only performs the stages you enable.

```
tasks/main.yml
│
├── update_hosts.yml       always runs (unless update_hosts: false)
├── checks.yml             runs when: preflight_checks: true
├── format_mount.yml       runs when: format_and_mount: true
├── packages.yml           runs when: packages_stage: true
├── install.yml            runs when: install_es: true
├── configure.yml          runs when: es_node_type is defined
└── upgrade.yml            runs when: rolling_upgrade: true
```

### Stage details

#### `update_hosts.yml` — `/etc/hosts` management
Builds a list of all cluster nodes (from `es_master`, `es_data`, and `es_all_in_one` groups) and
writes a managed block into `/etc/hosts` on every node using `blockinfile`. This allows nodes to
resolve each other by hostname without requiring DNS.

The block is marked with `# BEGIN/END ANSIBLE ELASTICSEARCH INVENTORY` so it is idempotent and
safe to re-run. To skip it, set `update_hosts: false`.

#### `checks.yml` — Preflight assertions
Runs three checks before any destructive work:

1. **Inventory check** — asserts that the inventory contains either `[es_all_in_one]` (at least
   one host) or both `[es_master]` and `[es_data]` (at least one host each). Fails fast with a
   clear message if neither condition is true.
2. **OS check** — asserts that each target host runs Debian or Ubuntu.
3. **Disk check** — on data-bearing nodes (`es_data` or `es_all_in_one`), stats the configured
   `es_data_disk` path and fails if it does not exist.

Enable with `preflight_checks: true`. Recommended for first runs.

#### `format_mount.yml` — Block device preparation
Runs only on data-bearing nodes (`es_data` or `es_all_in_one`). Steps:

1. Stats `es_data_disk` — fails if the path is missing or is not a block device.
2. Runs `blkid` to detect whether a filesystem already exists on the device.
3. Runs `findmnt` to detect whether the device is already mounted.
4. **Safety gate** — if the device is currently mounted at a location other than `es_mount_point`,
   the task fails immediately and requires manual intervention. This prevents accidental data loss.
5. Creates a filesystem (`es_filesystem`, default `xfs`) only if no existing filesystem is
   detected and `es_format_disks: true`.
6. Ensures the mount point directory exists.
7. Retrieves the device UUID via `blkid`.
8. Mounts the device by UUID (more stable than device path) and writes an `fstab` entry with
   `defaults,noatime,nodiratime` mount options for better I/O performance.

#### `packages.yml` — APT prerequisites and Elastic repository
Installs `apt-transport-https`, `ca-certificates`, `gnupg`, and `wget`. When
`es_install_method: repo`:

1. Downloads the Elastic GPG key to `/tmp/elastic.gpg`.
2. De-armors the key into `elastic_keyring_path` using `gpg --dearmor` (idempotent — skipped if
   the keyring file already exists).
3. Adds the Elastic APT repository file with the `signed-by` option pointing to the keyring.
4. Runs `apt update`.

When `es_install_method: deb`, this stage copies the local `.deb` file from `files/` to `/tmp/`
on the target host (no repo setup needed).

#### `install.yml` — Elasticsearch package installation
Installs Elasticsearch using the chosen method:
- `repo` — `apt install elasticsearch={{ es_elasticsearch_version }}`
- `deb` — `apt install /tmp/{{ es_deb_filename }}`

#### `configure.yml` — Node configuration
Gathers facts, computes JVM heap, renders templates, and starts the service. Behavior depends on
`es_node_type`. See [Node Types and configure.yml](#11-node-types-and-configureyml) for full
detail.

#### `upgrade.yml` — Rolling per-node upgrade
Runs only when `rolling_upgrade: true`. Performs the official Elastic rolling-upgrade procedure on
a **single** node. It is meant to be driven by a play with `serial: 1` so the cluster is upgraded
one node at a time with zero downtime. See [Rolling Upgrades](#12-rolling-upgrades-major-version)
for the full procedure and constraints.

---

## 4. Variables Reference

All defaults live in `defaults/main.yml` and can be overridden in inventory, group vars, or
as `vars:` in the playbook.

### Cluster identity

| Variable | Default | Description |
|---|---|---|
| `es_cluster_name` | `"my-es-cluster"` | Written into `cluster.name` in `elasticsearch.yml`. All nodes in the same cluster must share this value. |

### Installation method

| Variable | Default | Description |
|---|---|---|
| `es_install_method` | `"repo"` | `"repo"` uses the official Elastic APT repository. `"deb"` copies and installs a local `.deb` file. |
| `es_deb_filename` | `""` | Filename of the local `.deb` inside `files/`. Only required when `es_install_method: "deb"`. |
| `es_elasticsearch_version` | `"9.2.2"` | Package version pinned when installing from the APT repo. |

### APT repository (repo method only)

| Variable | Default | Description |
|---|---|---|
| `es_repo_url` | `"https://artifacts.elastic.co/packages/9.x/apt"` | Base URL for the Elastic APT repository. |
| `es_key_url` | `"https://artifacts.elastic.co/GPG-KEY-elasticsearch"` | URL of the Elastic GPG signing key. |
| `elastic_keyring_path` | `"/usr/share/keyrings/elasticsearch-keyring.gpg"` | Path where the de-armored GPG keyring is written. Referenced in the APT sources line. |
| `es_repo_list` | `"/etc/apt/sources.list.d/elastic-9.x.list"` | Path of the APT source file created by the role. |

### Network

| Variable | Default | Description |
|---|---|---|
| `es_http_port` | `9200` | Written into `http.port` in `elasticsearch.yml`. |

### Disk and mount

| Variable | Default | Description |
|---|---|---|
| `es_data_disk` | `"/dev/sdb"` | Block device to format and mount. **Override this per host** — the default is a placeholder. |
| `es_mount_point` | `"/var/lib/elasticsearch"` | Directory where the device is mounted. Elasticsearch's `path.data` is set to this value. |
| `es_filesystem` | `"xfs"` | Filesystem type created on `es_data_disk`. XFS is recommended for Elasticsearch data directories. |
| `es_format_disks` | `true` | Whether to format `es_data_disk` when no existing filesystem is detected. Set to `false` to skip formatting (safe mode). |
| `es_mount_opts` | `"defaults,noatime,nodiratime"` | Mount options written to fstab. `noatime`/`nodiratime` reduce unnecessary inode writes. |

### JVM

| Variable | Default | Description |
|---|---|---|
| `es_max_jvm_mb` | `30720` | Cap for the computed JVM heap in MB (~30 GB). The role will never recommend a heap larger than this value, regardless of available RAM. |
| `es_tmpdir` | `"/tmp"` | Written as `-Djava.io.tmpdir` in the JVM options. |

### Upgrade

| Variable | Default | Description |
|---|---|---|
| `rolling_upgrade` | `false` | When `true`, the role runs `upgrade.yml` (rolling per-node upgrade) instead of a fresh install. Drive it from a play with `serial: 1`. |
| `es_deb_filename` | `""` | For the `deb` method, set this to the **target** version's `.deb` in the upgrade play's `vars:` (e.g. `elasticsearch-9.2.2-amd64.deb`). |
| `es_elasticsearch_version` | `"9.2.2"` | For the `repo` method, set this to the target version in the upgrade play's `vars:`. |

See [Rolling Upgrades](#12-rolling-upgrades-major-version) for the full procedure.

---

## 5. Topology A — Dedicated Masters + Data Nodes

Use this topology when you want strict role separation. Master nodes handle cluster coordination
exclusively; data nodes store indices and handle search/indexing.

### Inventory (`hosts.ini`)

```ini
[es_master]
es-master-1 ansible_host=192.168.3.221
es-master-2 ansible_host=192.168.3.222
es-master-3 ansible_host=192.168.3.223

[es_data]
es-data-1 ansible_host=192.168.3.231
es-data-2 ansible_host=192.168.3.232
es-data-3 ansible_host=192.168.3.233

[all:vars]
ansible_user=ali
```

Three masters satisfy the quorum requirement (majority = 2). Three data nodes give you resilience
against a single node failure.

### Execution order

The plays in `site.yml` run in this exact order:

```
Play 1: Base preparation  →  all masters + all data nodes
         (preflight_checks, format_and_mount, packages_stage)

Play 2: Bootstrap         →  es-master-1 only
         Starts the cluster. Sets cluster.initial_master_nodes.
         node.roles = [master]

Play 3: Master nodes      →  es-master-2, es-master-3
         Joins the already-bootstrapped cluster.
         node.roles = [master]

Play 4: Data nodes        →  es-data-1, es-data-2, es-data-3
         Joins the cluster and stores data.
         node.roles = [data]
```

**Why this order matters:** Elasticsearch's cluster bootstrap (`cluster.initial_master_nodes`)
must only be set on the very first startup of the cluster. Once the bootstrap master has started
and the cluster is formed, the remaining masters and data nodes join by discovering seed hosts
(`discovery.seed_hosts`). If `cluster.initial_master_nodes` were present on a node restarting
into an already-formed cluster, it would be ignored — but setting it on Play 3/4 is still wrong
practice and can cause split-brain in edge cases. The role ensures only the bootstrap node (Play 2)
carries this setting.

### Run command

```bash
# Run all stages in the correct order
ansible-playbook -i hosts.ini site.yml

# Run only base preparation (safe to re-run)
ansible-playbook -i hosts.ini site.yml --tags "" --limit es_master:es_data

# Run only a specific play by name
ansible-playbook -i hosts.ini site.yml --start-at-task "Bootstrap cluster"
```

---

## 6. Topology B — All-in-One Nodes

Use this topology when you want every node to participate in both master election and data storage.
Common for smaller clusters, dev/staging environments, or when you don't need role separation.

Every node carries `node.roles: [master, data]`. The cluster still bootstraps with a single
designated bootstrap node; the rest join it.

### Inventory (`hosts.ini`)

```ini
[es_all_in_one]
es-node-1 ansible_host=192.168.3.211
es-node-2 ansible_host=192.168.3.212
es-node-3 ansible_host=192.168.3.213

[all:vars]
ansible_user=ali
```

The bootstrap node is always `groups['es_all_in_one'][0]` — the first host listed in the group.
There is no hardcoded hostname; changing the order in the inventory changes which node bootstraps.

### Execution order

```
Play 5: Base preparation (all-in-one)  →  all es_all_in_one nodes
         (preflight_checks, format_and_mount, packages_stage)

Play 6: Bootstrap all-in-one cluster  →  es_all_in_one[0] only
         Starts the cluster. Sets cluster.initial_master_nodes.
         node.roles = [master, data]

Play 7: All-in-one remaining nodes     →  es_all_in_one minus the first
         Joins the already-bootstrapped cluster.
         node.roles = [master, data]
```

### Run command

```bash
# Run only the all-in-one topology plays
ansible-playbook -i hosts.ini site.yml --limit es_all_in_one
```

### Key differences from Topology A

| | Dedicated | All-in-one |
|---|---|---|
| `node.roles` | `[master]` or `[data]` | `[master, data]` on every node |
| Disk formatting | Only on `es_data` nodes | On all `es_all_in_one` nodes |
| `/etc/hosts` | `es_master` + `es_data` entries | `es_all_in_one` entries |
| `discovery.seed_hosts` | `groups['es_master']` | `groups['es_all_in_one']` |
| `cluster.initial_master_nodes` | `groups['es_master']` | `groups['es_all_in_one']` |

---

## 7. Installation Methods

### Method 1: `repo` (default)

Installs Elasticsearch from the official Elastic APT repository. Requires the target hosts to
have internet access (or a proxied/mirrored repository at `es_repo_url`).

```yaml
es_install_method: "repo"
es_elasticsearch_version: "9.2.2"
```

Steps the role performs:
1. Installs APT prerequisites (`apt-transport-https`, `ca-certificates`, `gnupg`, `wget`).
2. Downloads the Elastic GPG public key from `es_key_url` into `/tmp/elastic.gpg`.
3. De-armors the key into `elastic_keyring_path` with `gpg --dearmor` (idempotent).
4. Writes an APT source file referencing the keyring:
   ```
   deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg]
       https://artifacts.elastic.co/packages/9.x/apt stable main
   ```
5. Runs `apt update` and installs `elasticsearch=9.2.2`.

### Method 2: `deb` (local package)

Installs from a `.deb` file you provide. Useful for air-gapped environments.

```yaml
es_install_method: "deb"
es_deb_filename: "elasticsearch-9.2.2-amd64.deb"
```

Steps the role performs:
1. Copies `files/elasticsearch-9.2.2-amd64.deb` from the role to `/tmp/` on each target host.
2. Installs it with `apt install /tmp/elasticsearch-9.2.2-amd64.deb`.

Place the `.deb` file in `elasticsearch_cluster/files/` before running. The file is listed in
`.gitignore` because Debian packages are large binaries that do not belong in version control.

---

## 8. Disk Formatting and Mounting

Controlled by `format_and_mount: true` in the play vars. Runs on all data-bearing nodes
(`es_data` in Topology A, `es_all_in_one` in Topology B).

### What the role does

1. **Block device check** — Stats `es_data_disk`. Fails if the path does not exist or is not a
   block device. This prevents silent misconfiguration.

2. **Filesystem detection** — Runs `blkid -s TYPE` to check whether the device already has a
   filesystem. If it does, formatting is skipped entirely regardless of `es_format_disks`.

3. **Mount location safety check** — Runs `findmnt` to see if the device is already mounted. If
   it is mounted at a location *other than* `es_mount_point`, the task fails and requires manual
   intervention. This prevents accidental data destruction.

4. **Formatting** — If no filesystem is detected and `es_format_disks: true`, creates an
   `es_filesystem` filesystem on the device. If `es_format_disks: false`, this step is skipped
   even when no filesystem exists, allowing you to prepare the device manually.

5. **Mount point creation** — Ensures the `es_mount_point` directory exists with `0755`
   permissions.

6. **UUID-based mount** — Retrieves the device UUID with `blkid -s UUID` and mounts by UUID
   (e.g., `UUID=abc123...`). UUID-based mounts are stable across reboots even if device paths
   change (e.g., `/dev/sdb` becoming `/dev/sdc`).

7. **fstab entry** — Writes a persistent mount entry using `defaults,noatime,nodiratime`.
   `noatime` prevents the kernel from updating file access timestamps on every read, which reduces
   unnecessary disk writes on Elasticsearch workloads.

### Recommended variable overrides

```yaml
# Override per host in host_vars/ or in the inventory
es_data_disk: "/dev/nvme1n1"    # always be explicit — /dev/sdb is just a placeholder
es_mount_point: "/data/elasticsearch"
es_filesystem: "xfs"
es_format_disks: false          # set false if you pre-format the disk manually
```

---

## 9. JVM Heap Sizing

The role automatically computes an appropriate JVM heap size for each node and writes it into
`/etc/elasticsearch/jvm.options.d/heap.options`. This file sets both `-Xms` (initial heap) and
`-Xmx` (maximum heap) to the same value, which is the recommended practice for Elasticsearch
to avoid heap resizing pauses at runtime.

### Calculation

```
es_recommended_jvm_mb = min(ansible_memtotal_mb / 2, es_max_jvm_mb)
```

- **Half of RAM** — Elasticsearch benefits from a large heap, but the other half of RAM should
  remain available to the OS page cache. Lucene (the search library Elasticsearch uses internally)
  heavily relies on the OS page cache for index reads; starving it of memory degrades search
  performance significantly.
- **Cap at `es_max_jvm_mb` (default 30 GB)** — Java's compressed ordinary object pointers (OOPs)
  are enabled when the heap is below approximately 32 GB, allowing the JVM to use 32-bit
  references for objects and reducing memory overhead. Heaps above ~31–32 GB cross this threshold
  and lose compressed OOPs, which can increase memory usage and reduce throughput. The default
  cap of 30720 MB keeps the heap safely within the compressed-OOP range.

### JVM options written

```
-Xms<computed>m
-Xmx<computed>m
-XX:+UseG1GC
-Djava.io.tmpdir=/tmp
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
-Xlog:gc*,...
```

G1GC is the recommended garbage collector for Elasticsearch 8+. `HeapDumpOnOutOfMemoryError` and
`ExitOnOutOfMemoryError` are standard for production: they produce a heap dump for post-mortem
analysis and immediately terminate the process (rather than running in a degraded state) on OOM.

### Changing the cap

```yaml
# In group_vars or playbook vars
es_max_jvm_mb: 16384    # cap at 16 GB on smaller nodes
```

---

## 10. Elasticsearch Configuration Template

The role renders `elasticsearch.yml` from `templates/elasticsearch.yml.j2`. The template accepts
variables passed from `configure.yml` via `vars:` on the template task.

### Generated file structure

```yaml
cluster.name: "my-es-cluster"
node.name: "es-master-1"          # always set to inventory_hostname

node.roles: ["master"]             # or ["data"], or ["master", "data"]

path.data: "/var/lib/elasticsearch"
path.logs: /var/log/elasticsearch

network.host: 0.0.0.0
http.port: 9200

discovery.seed_hosts: ["es-master-1", "es-master-2", "es-master-3"]

# Only present on the bootstrap node:
cluster.initial_master_nodes: ["es-master-1", "es-master-2", "es-master-3"]

xpack.security.enabled: false
xpack.security.transport.ssl.enabled: false
xpack.security.http.ssl.enabled: false
```

### Template variables

| Variable | Who sets it | Purpose |
|---|---|---|
| `node_roles` | `configure.yml` per node type | List written to `node.roles` |
| `seed_hosts` | `configure.yml` per topology | List written to `discovery.seed_hosts` |
| `include_initial_master` | `configure.yml`, `true` only on bootstrap node | Controls whether `cluster.initial_master_nodes` appears |
| `initial_master_nodes` | `configure.yml`, bootstrap only | Value of `cluster.initial_master_nodes` |

### Security note

`xpack.security.enabled: false` disables authentication and TLS. This is suitable for internal
networks and development environments. For production, enable security, generate TLS certificates,
and configure authentication before exposing the cluster on any network.

---

## 11. Node Types and `configure.yml`

The `configure.yml` task file is selected when `es_node_type` is defined. It handles five node
types, each corresponding to a block in the file:

| `es_node_type` | Topology | `node.roles` | `cluster.initial_master_nodes` |
|---|---|---|---|
| `bootstrap` | Dedicated | `[master]` | Set to all `es_master` hosts |
| `master` | Dedicated | `[master]` | Not set |
| `data` | Dedicated | `[data]` | Not set |
| `bootstrap_all_in_one` | All-in-one | `[master, data]` | Set to all `es_all_in_one` hosts |
| `all_in_one` | All-in-one | `[master, data]` | Not set |

In all cases `configure.yml`:
1. Ensures Ansible facts are available (runs `setup` if not already gathered).
2. Computes `es_recommended_jvm_mb` and `es_seed_hosts` via `set_fact`.
3. Creates `/etc/elasticsearch` and `/etc/elasticsearch/jvm.options.d` directories.
4. Renders the JVM heap options file.
5. Fixes ownership of `es_mount_point` to `elasticsearch:elasticsearch`.
6. Renders `elasticsearch.yml` with the correct `node_roles` and discovery settings.
7. Notifies the `restart elasticsearch` handler on any config change.
8. Ensures `elasticsearch.service` is enabled and started.

The handler (`handlers/main.yml`) runs `systemctl daemon-reload` then restarts the service, so
changes to configuration take effect without manual intervention.

---

## 12. Rolling Upgrades (Major Version)

The role can upgrade an existing cluster in place, one node at a time, with **zero downtime**.
This is driven by `tasks/upgrade.yml` and gated by the `rolling_upgrade: true` flag. A reference
playbook for the all-in-one topology lives at `test/upgrade.yml`.

### Supported upgrade path (read first)

Elasticsearch enforces a strict major-version upgrade path. **You cannot jump from an arbitrary
8.x straight to 9.1+.** You must first be on the final 8.x minor (**8.19.x**), then upgrade to 9.x.

| Current version | Target | Allowed directly? |
|---|---|---|
| 8.17.x or earlier | 9.x | ❌ — upgrade to 8.19.x first |
| 8.18.x | 9.0.x | ✅ (but not 9.1+) |
| **8.19.x** | **9.0 – 9.2** | ✅ |

Other compatibility notes:
- **Indices created in 7.x or earlier** must be reindexed, deleted, or archived before upgrading
  to 9.x — incompatible indices block node startup.
- The v1 legacy `_template` API is removed in 9.x; migrate to composable index templates first.
- Plugins compiled for 8.x will not load on 9.x.
- 8.x clients keep working against a 9.x cluster via REST API compatibility.

### A major upgrade is NOT reversible

A major upgrade rewrites the on-disk data directory format. **There is no downgrade** by
reinstalling the old binary. Your only rollback is **snapshot → restore into a fresh old-version
cluster**. Always take a snapshot before upgrading a real cluster.

### What `upgrade.yml` does per node

Run from a play with `serial: 1`, the role visits one node at a time and performs:

| Step | Action |
|---|---|
| 1 | Copy the target `.deb` to `/tmp/` (deb method only) |
| 2 | Wait for the local ES API to be reachable |
| 3 | Disable cluster-wide shard allocation (`cluster.routing.allocation.enable: primaries`) |
| 4 | Flush indices to speed up shard recovery |
| 5 | Stop Elasticsearch on this node |
| 6 | Install the target package in place (`apt` deb or pinned repo version) |
| 7 | Start Elasticsearch and wait for the node to rejoin |
| 8 | Re-enable shard allocation (reset to default) |
| 9 | Wait for the cluster to return to **green** before the play moves to the next node |
| 10 | Report the running version on the node |

The disable/enable allocation calls are cluster-wide and are issued against the node's own HTTP
API (bound to `es_network_host`). Because each node is upgraded only after the cluster is green
again, and shards have replicas on the other nodes, the cluster keeps serving throughout.

### Node order

Upgrade **master-eligible nodes last**. In the all-in-one topology every node is master+data, so
order does not matter and inventory order is fine. In the dedicated topology, upgrade data nodes
first and master nodes last.

### Running an upgrade

Point the install inventory at the current (lower) version, then run a dedicated upgrade play that
overrides the target version. Example (all-in-one, deb method):

```yaml
# upgrade.yml
- name: Rolling upgrade Elasticsearch 8.19.x -> 9.2.2 (all-in-one)
  hosts: es_all_in_one
  become: true
  serial: 1                  # one node at a time
  max_fail_percentage: 0     # abort the whole run if any node fails to go green
  vars:
    update_hosts: false
    rolling_upgrade: true
    es_deb_filename: elasticsearch-9.2.2-amd64.deb   # the TARGET version
  roles:
    - role: elasticsearch_cluster
```

```bash
ansible-playbook -i hosts.ini upgrade.yml
```

`max_fail_percentage: 0` combined with `serial: 1` means that if any single node fails to return
to green within the timeout, the play aborts and the remaining nodes stay on the old version,
giving you a chance to investigate before continuing. Because the deb-install step is idempotent
(it is a no-op once a node is already on the target version), re-running the playbook after fixing
a problem safely resumes the upgrade.

### Verifying the result

```bash
# All nodes should report the target version
curl -s "http://<node-ip>:9200/_cat/nodes?v=true&h=name,version,node.role"

# Cluster should be green; any test data should be intact
curl -s "http://<node-ip>:9200/_cluster/health?pretty"
```

> A full, reproducible upgrade test (install 8.19.17 → load data → rolling-upgrade to 9.2.2 on
> three Vagrant VMs) is documented in [`test/README.md`](test/README.md).

---

## 13. Safety and Operational Cautions

### Disk formatting is destructive
`format_mount.yml` will create a new filesystem on `es_data_disk` if no filesystem is detected
and `es_format_disks: true`. Always verify `es_data_disk` is set to the correct path before
running with `format_and_mount: true`. The default value `/dev/sdb` is a placeholder — override
it in `host_vars/` or in the inventory.

### `network.host: 0.0.0.0`
The role binds Elasticsearch to all interfaces. This is convenient for initial setup but is
permissive. For production, restrict binding to the specific interface/IP and configure firewall
rules to allow traffic only on ports 9200 (HTTP) and 9300 (transport) from trusted hosts.

### Security is disabled
`xpack.security.enabled: false` means no authentication, no encryption. Suitable only for
closed internal networks. Enable security before any exposure to untrusted networks.

### `cluster.initial_master_nodes` and re-runs
The `cluster.initial_master_nodes` setting is only meaningful during initial cluster bootstrap.
Once the cluster is formed, this setting is ignored by Elasticsearch on subsequent restarts. The
role correctly sets it only on the bootstrap node play. Do not set it manually on existing
clusters — it does not re-initialize the cluster but it is confusing and should not be present.

### `gather_facts` must be enabled
The JVM heap calculation uses `ansible_memtotal_mb`. Plays in `site.yml` rely on Ansible's
implicit fact gathering (enabled by default). If you add plays with `gather_facts: false`, set
`es_max_jvm_mb` explicitly or the JVM options render will fail.

### Fact requirement for `getent_passwd`
The ownership task at the end of `configure.yml` is gated on
`ansible_facts.getent_passwd.elasticsearch is defined`. This fact is populated when the
`gather_subset: ['!all', 'min', 'getent_passwd']` is included in the gather. If running with
minimal fact gathering, the ownership task is skipped silently. Run `format_mount.yml` first
(which creates the directory) and verify ownership manually if needed.
