
## Purpose
This Ansible role installs and configures a simple Elasticsearch cluster (master and data nodes). It supports two installation methods — `repo` (official Elastic APT repository) and `deb` (local Debian package) — and includes optional block-device formatting and mounting for data nodes.


---

## Key behavior (what the role does)

### 1. High-level task flow
When executed with the variables in `site.yml` or the role-level variables, the role performs the following logical stages (each stage is included by `tasks/main.yml` and gated by feature flags / vars):

- `update_hosts.yml` (optional): build `/etc/hosts` entries from the inventory (this is run with `run_once` by default in the role).
- `checks.yml` (optional, enabled via `preflight_checks: true`): inventory and OS checks (requires at least one `es_master` and one `es_data` host; ensures Debian/Ubuntu).
- `format_mount.yml` (optional, enabled via `format_and_mount: true`): on data nodes formats a block device and mounts it at `es_mount_point` (default `/var/lib/elasticsearch`) using `es_filesystem` (default `xfs`).
- `packages.yml` (optional, enabled via `packages_stage: true`): installs apt prerequisites and configures the Elastic APT repository (when `es_install_method: repo`).
- `install.yml` (optional, enabled via `install_es: true`): installs Elasticsearch either from the APT repo or from a local `.deb`.
- `configure.yml` (enabled when `es_node_type` is set): renders JVM options and `elasticsearch.yml` templates for bootstrap/master/data roles, fixes ownerships and ensures the service is enabled+started.

---

## Important variables (defaults and meaning)
Key variables are defined in `defaults/main.yml`:

- `es_cluster_name` (default: `"my-es-cluster"`) — cluster name written into `elasticsearch.yml`.
- `es_install_method` (default: `"repo"`) — `"repo"` uses the official Elastic APT repo; `"deb"` copies and installs a local `.deb`.
- `es_deb_filename` (default: `""`) — name of the local `.deb` when `es_install_method: "deb"`. The role copies it to `/tmp/{{ es_deb_filename }}` on the target and installs it with `apt`.
- `es_http_port` (default: `9200`) — HTTP port.
- `es_data_disk` (default: `"/dev/sdb"`) — block device used for data (used by `format_and_mount` tasks).
- `es_mount_point` (default: `"/var/lib/elasticsearch"`) — mount path for the data device.
- `es_filesystem` (default: `"xfs"`) — filesystem to create on `es_data_disk` when formatting.
- `es_format_disks` (default: `true`) — whether to format the device if no filesystem is detected.
- `es_elasticsearch_version` (default: `"9.2.2"`) — package version installed from APT when `es_install_method: repo`.
- `elastic_keyring_path` (default: `"/usr/share/keyrings/elasticsearch-keyring.gpg"`) — where the dearmored GPG key is written and referenced by the apt repository line.
- `es_max_jvm_mb` (default: `30720`) — the configured cap for the JVM heap (in MB). The role computes `es_recommended_jvm_mb` as `min(ansible_memtotal_mb/2, es_max_jvm_mb)` and writes this into the `jvm.options.d/heap.options` file.

> The role's JVM-calc behavior: it uses half of the host RAM (rounded down) but will not exceed `es_max_jvm_mb`. This default cap is 30720 MB (~30 GB).

---

## XFS and filesystem guidance
- The role defaults to creating `xfs` on the `es_data_disk` and mounting it at `es_mount_point`. XFS is commonly recommended for Elasticsearch data directories in large production environments; the community and operational guides frequently prefer XFS for heavy-data workloads and recommend tuning mount options (noatime, appropriate readahead, etc.) when operating at scale. :contentReference[oaicite:0]{index=0}

**Operational note / caution**
- The role will format the device if it detects no filesystem and `es_format_disks: true`. Formatting a device is destructive — **verify the device path and contents before running** (consider setting `es_format_disks: false` and running the role after preparing the device manually).
- For more robust and stable mounts it is recommended to mount by UUID (the current role uses the device path configured in `es_data_disk`).

---

## JVM heap sizing (why the default cap)
- The role caps recommended heap at `es_max_jvm_mb` (default 30720 MB) and computes `es_recommended_jvm_mb = min(ansible_memtotal_mb/2, es_max_jvm_mb)`. This follows common guidance to:
  - Give Elasticsearch a large heap but keep it under the Java compressed-object-pointer threshold (close to 32 GB) for optimal performance. The 30–31 GB range is commonly used in production to retain compressed OOPs and avoid the performance penalty that can appear when heap grows beyond the compressed-oop threshold. :contentReference[oaicite:1]{index=1}

**Recommendation**
- Verify the computed heap (`es_recommended_jvm_mb`) for each host after running the role. For production nodes consider leaving headroom for the filesystem cache (the usual heuristic is not to allocate more than ~50% of available RAM to the JVM heap). :contentReference[oaicite:2]{index=2}

---

## Install methods — `repo` vs `deb` (how this role implements them)

### `repo` (default)
- Behavior:
  1. The role installs apt prerequisites (`apt-transport-https`, `gnupg`, etc.).
  2. It downloads the Elastic GPG key to `/tmp/elastic.gpg`, then de-armors it into `{{ elastic_keyring_path }}` using `gpg --dearmor`.
  3. It creates an APT repository file containing the signed-by option that points to the keyring:
     ```
     deb [signed-by={{ elastic_keyring_path }}] {{ es_repo_url }} stable main
     ```
  4. The role runs `apt update` and installs `elasticsearch={{ es_elasticsearch_version }}` from the repository.

- When to use:
  - Use this in environments where you can reach `https://artifacts.elastic.co/` (i.e., hosts have internet access or you have a proxied repository). This is the recommended method for straightforward upgrades and receiving package updates. :contentReference[oaicite:3]{index=3}

### `deb` (local package)
- Behavior:
  1. The playbook variable `es_deb_filename` is read (must be set).
  2. The role copies the file `files/{{ es_deb_filename }}` from the role to `/tmp/{{ es_deb_filename }}` on the target host.
  3. The role installs it with `apt: deb=/tmp/{{ es_deb_filename }}`.

- Where to put the `.deb`:
  - Place the `.deb` file inside the role’s `files/` directory (i.e. `roles/elasticsearch_cluster/files/<your-elasticsearch-package>.deb`) and set:
    ```yaml
    es_install_method: "deb"
    es_deb_filename: "elasticsearch-9.2.2-amd64.deb"
    ```
  - Alternatively, you may make the `.deb` accessible through any mechanism Ansible `copy` supports (local path in the control machine `files/` is the normal pattern). The role then copies that file to `/tmp` on the managed host and installs it with `apt`.
- Note:
  - Because `.deb` files are often large and proprietary you may prefer to keep them out of Git (see `.gitignore` above). Use a private artifact repository or the `repo` method where possible.

---

## Safety & operational cautions (summary)
- **Formatting is destructive.** Confirm `es_data_disk` before running with `format_and_mount: true`.
- The default `es_data_disk` is `/dev/sdb` in `defaults/main.yml`. It is strongly recommended to override this per-host and not rely on the default.
- The role computes heap using `ansible_memtotal_mb`. Ensure facts are gathered or run with `gather_facts: true` on the play.
- The role writes JVM options to `/etc/elasticsearch/jvm.options.d/heap.options`. If you maintain other JVM options, review for conflicts.
- `network.host` in the role template is set to `0.0.0.0`. That is permissive — adjust for production (bind to an explicit interface/IP or enable firewall rules and TLS/authentication).

---

## Example usage (playbook excerpt)
```yaml
- hosts: es_master:es_data
  become: true
  roles:
    - role: elasticsearch_cluster
      vars:
        preflight_checks: true
        format_and_mount: true
        packages_stage: true

# bootstrap master
- hosts: es-master-1
  become: true
  roles:
    - role: elasticsearch_cluster
      vars:
        install_es: true
        es_node_type: bootstrap
