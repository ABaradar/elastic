# Vagrant Test Environment

Local 3-node Elasticsearch cluster running on VirtualBox VMs for testing the
`elasticsearch_cluster` Ansible role.

## What gets created

| VM | Private IP | RAM | Extra disk |
|---|---|---|---|
| es-node-1 | 192.168.56.211 | 3 GB | 20 GB VDI → `/dev/sdb` |
| es-node-2 | 192.168.56.212 | 3 GB | 20 GB VDI → `/dev/sdb` |
| es-node-3 | 192.168.56.213 | 3 GB | 20 GB VDI → `/dev/sdb` |

All three nodes use the **all-in-one topology** — each node carries both `master`
and `data` roles. Elasticsearch is installed from a local `.deb` file (no internet
required on the VMs).

This environment supports two scenarios:

1. **Fresh install** of a 3-node cluster (`site.yml`).
2. **Rolling major upgrade** from **8.19.17 → 9.2.2** with zero downtime
   (`upgrade.yml`). See [Testing the 8 → 9 rolling upgrade](#testing-the-8--9-rolling-upgrade).

---

## Prerequisites

- Vagrant ≥ 2.4
- VirtualBox ≥ 7.1
- Ansible installed on your control machine
- The box already downloaded:
  ```bash
  vagrant box add bento/ubuntu-24.04
  ```
- The `.deb` packages present in `../elasticsearch_cluster/files/`:
  - `elasticsearch-8.19.17-amd64.deb` (initial install — the required 8.19.x stepping stone)
  - `elasticsearch-9.2.2-amd64.deb` (upgrade target)

  Download them with:
  ```bash
  cd ../elasticsearch_cluster/files/
  curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.19.17-amd64.deb
  curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.2.2-amd64.deb
  ```

---

## Step-by-step usage

All commands below are run from the `test/` directory.

### 1. Start the VMs

```bash
cd test/
vagrant up
```

This creates three VMs, attaches a 20 GB disk to each, and boots them. First run
takes a few minutes. VDI files land in `test/.vagrant/disks/` and are git-ignored.

**If `vagrant up` fails** with a storage controller error like
`Could not find a controller named 'SATA Controller'`, run:

```bash
VBoxManage showvminfo es-node-1 | grep "Storage Controller"
```

Copy the exact controller name shown and update `--storagectl` in the Vagrantfile.

### 2. Verify SSH connectivity

```bash
ansible all -m ping
```

Expected output: three `SUCCESS` pings.

### 3. Run the Ansible playbook

```bash
ansible-playbook site.yml
```

The playbook runs three plays in order:

| Play | Hosts | What happens |
|---|---|---|
| Base preparation | all 3 nodes | Preflight checks, format + mount `/dev/sdb`, copy `elasticsearch-8.19.17-amd64.deb` to `/tmp/` |
| Bootstrap cluster | es-node-1 only | Install ES 8.19.17, write `elasticsearch.yml` with `cluster.initial_master_nodes`, start service |
| Remaining nodes | es-node-2, es-node-3 | Install ES 8.19.17, write `elasticsearch.yml`, join the cluster |

### 4. Verify the cluster

> **Note:** each node binds its HTTP API to its **private IP** (`es_network_host`),
> not `localhost`. Use the node's `192.168.56.x` address for every `curl` — this
> works both from your host and from inside the VM.

```bash
curl -s http://192.168.56.211:9200/_cluster/health?pretty
```

Expected: `"status": "green"`, `"number_of_nodes": 3`.

Check node roles and versions:

```bash
curl -s http://192.168.56.211:9200/_cat/nodes?v=true&h=name,version,node.role
```

Each node should report version `8.19.17`.

---

## Testing the 8 → 9 rolling upgrade

The `upgrade.yml` playbook performs a **zero-downtime rolling upgrade** from
8.19.17 to 9.2.2, one node at a time.

### Why 8.19.x first?

Elasticsearch enforces a supported upgrade path: **you cannot jump from an
arbitrary 8.x straight to 9.1+**. You must first be on **8.19.x**, then upgrade
to 9.x. That is exactly why the fresh install above lands on 8.19.17. A major
upgrade also **rewrites the data directory and is not reversible** — the only
real rollback is snapshot + restore into a fresh old cluster. For this throwaway
test cluster, the rollback is simply `vagrant destroy` and reprovision.

### 1. (Recommended) Load some data first

An empty cluster goes green instantly, so it doesn't really exercise shard
recovery. Create an index with a replica and a few documents so the upgrade has
shards to relocate and you can confirm nothing is lost:

```bash
# Create an index with 1 replica so every shard exists on 2 of the 3 nodes
curl -s -XPUT "http://192.168.56.211:9200/upgrade-test" \
  -H 'Content-Type: application/json' \
  -d '{"settings":{"number_of_shards":3,"number_of_replicas":1}}'

# Index a few documents
for i in 1 2 3 4 5; do
  curl -s -XPOST "http://192.168.56.211:9200/upgrade-test/_doc" \
    -H 'Content-Type: application/json' \
    -d "{\"msg\":\"doc $i\",\"ts\":\"$(date -Iseconds)\"}" >/dev/null
done

# Confirm the count
curl -s "http://192.168.56.211:9200/upgrade-test/_count?pretty"
```

### 2. Run the rolling upgrade

```bash
ansible-playbook upgrade.yml
```

What happens, **per node, strictly one at a time** (`serial: 1`):

| Step | Action |
|---|---|
| 1 | Copy `elasticsearch-9.2.2-amd64.deb` to `/tmp/` |
| 2 | Disable cluster-wide shard allocation (`cluster.routing.allocation.enable: primaries`) |
| 3 | Flush indices |
| 4 | Stop Elasticsearch on this node |
| 5 | `apt install` the 9.2.2 deb in place |
| 6 | Start Elasticsearch, wait for the node to rejoin |
| 7 | Re-enable shard allocation |
| 8 | Wait for the cluster to return to **green** before touching the next node |

If any node fails to come back green, `max_fail_percentage: 0` aborts the run so
the remaining nodes stay on the old version.

### 3. Keep the cluster reachable during the upgrade (optional)

In a second terminal, watch health while the upgrade runs. Because nodes are
upgraded one at a time and shards have replicas, the cluster keeps serving:

```bash
watch -n 2 'curl -s "http://192.168.56.212:9200/_cluster/health?pretty" | grep -E "status|number_of_nodes"'
```

Point it at a node that is **not** currently being upgraded (e.g. node-2 while
node-1 upgrades). The cluster may briefly show `yellow` while a node is down,
then return to `green`.

### 4. Verify the upgrade

```bash
# All nodes should now report 9.2.2
curl -s "http://192.168.56.211:9200/_cat/nodes?v=true&h=name,version,node.role"

# Data survived the upgrade
curl -s "http://192.168.56.211:9200/upgrade-test/_count?pretty"
```

Expected: three nodes on `9.2.2`, cluster `green`, and the document count
unchanged from before the upgrade.

---

## Re-running individual stages

Each play is gated by feature flags. You can re-run a single stage without
touching the rest.

```bash
# Re-run only configuration (e.g. after changing elasticsearch.yml.j2)
ansible-playbook site.yml --start-at-task "Configure node (render templates, owners, service)"

# Re-run only the install stage on all nodes
ansible-playbook site.yml -e "install_es=true" --skip-tags ""

# Run with verbose output
ansible-playbook site.yml -vv
```

---

## VM lifecycle

```bash
# Suspend all VMs (saves RAM while preserving state)
vagrant suspend

# Resume
vagrant resume

# Halt (clean shutdown)
vagrant halt

# Destroy everything (VMs + VDI disks are deleted)
vagrant destroy -f
```

---

## How install variables are set

The `deb` install method and related vars are set in `hosts.ini` under `[all:vars]`:

```ini
es_install_method=deb
es_deb_filename=elasticsearch-8.19.17-amd64.deb
es_data_disk=/dev/sdb
es_cluster_name=vagrant-test-cluster
```

`upgrade.yml` overrides `es_deb_filename` to `elasticsearch-9.2.2-amd64.deb` in
its play `vars`, so the initial install lands on 8.19.17 and the rolling upgrade
moves the cluster to 9.2.2.

These override the role defaults (`es_install_method: repo`) without touching the
main `site.yml` or `defaults/main.yml`, so they only apply to this test inventory.

The `.deb` is copied from the role's `files/` directory to `/tmp/` on each VM
once, then installed with `apt`. The `ansible.cfg` in this directory sets
`roles_path = ../` so Ansible finds the role at the project root.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `vagrant up` fails: controller not found | SATA controller has a different name on your box | Run `VBoxManage showvminfo es-node-1 \| grep "Storage Controller"` and update `Vagrantfile` |
| `ansible all -m ping` fails: SSH timeout | VMs not fully booted or wrong key path | Wait 30 s and retry; verify `.vagrant/machines/` contains the `private_key` files |
| ES won't start: `max virtual memory areas` error | `vm.max_map_count` too low | `vagrant ssh es-node-1 -- sudo sysctl -w vm.max_map_count=262144` (add to `/etc/sysctl.conf` for persistence) |
| Cluster stays yellow | Replicas can't be allocated with 3 nodes | Normal for default index settings; set `number_of_replicas: 0` on test indices |
| `.deb` copy is slow | ~648 MB file copied to each VM | Expected on first run; subsequent runs are idempotent and skip the copy if the file already exists in `/tmp/` |
| Upgrade aborts after first node | node didn't return to green within the timeout | Check `journalctl -u elasticsearch` on that node; a 7.x-created index or removed setting can block 9.x startup. Fix it, then re-run `upgrade.yml` (already-upgraded nodes are skipped at the deb-install step) |
| `curl localhost:9200` refused inside a VM | ES binds to the private IP (`es_network_host`), not loopback | Use the node's `192.168.56.x` address instead |
