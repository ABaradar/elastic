# Vagrant Test Environment

Local 3-node Elasticsearch cluster running on VirtualBox VMs for testing the
`elasticsearch_cluster` Ansible role.

## What gets created

| VM | Private IP | RAM | Extra disk |
|---|---|---|---|
| es-node-1 | 192.168.56.211 | 4 GB | 20 GB VDI → `/dev/sdb` |
| es-node-2 | 192.168.56.212 | 4 GB | 20 GB VDI → `/dev/sdb` |
| es-node-3 | 192.168.56.213 | 4 GB | 20 GB VDI → `/dev/sdb` |

All three nodes use the **all-in-one topology** — each node carries both `master`
and `data` roles. Elasticsearch is installed from a local `.deb` file (no internet
required on the VMs).

---

## Prerequisites

- Vagrant ≥ 2.4
- VirtualBox ≥ 7.1
- Ansible installed on your control machine
- The box already downloaded:
  ```bash
  vagrant box add bento/ubuntu-24.04
  ```
- The `.deb` package present at `../elasticsearch_cluster/files/elasticsearch-9.2.2-amd64.deb`

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
| Base preparation | all 3 nodes | Preflight checks, format + mount `/dev/sdb`, copy `.deb` to `/tmp/` |
| Bootstrap cluster | es-node-1 only | Install ES, write `elasticsearch.yml` with `cluster.initial_master_nodes`, start service |
| Remaining nodes | es-node-2, es-node-3 | Install ES, write `elasticsearch.yml`, join the cluster |

### 4. Verify the cluster

SSH into any node and check cluster health:

```bash
vagrant ssh es-node-1
curl -s http://localhost:9200/_cluster/health?pretty
```

Expected: `"status": "green"`, `"number_of_nodes": 3`.

Check node roles:

```bash
curl -s http://localhost:9200/_cat/nodes?v
```

From your host machine (using the VM's private IP):

```bash
curl -s http://192.168.56.211:9200/_cluster/health?pretty
```

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
es_deb_filename=elasticsearch-9.2.2-amd64.deb
es_data_disk=/dev/sdb
es_cluster_name=vagrant-test-cluster
```

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
| `.deb` copy is slow | 656 MB file copied to each VM | Expected on first run; subsequent runs are idempotent and skip the copy if the file already exists in `/tmp/` |
