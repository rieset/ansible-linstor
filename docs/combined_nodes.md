# Combined nodes in LINSTOR

## What is a Combined node?

**Combined node** = Controller + Satellite on one host

- Acts as controller (cluster management)
- Acts as satellite (data storage)
- Saves resources (no separate controller and satellite nodes)

## How to create a Combined node

**Add the host to both groups:**

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.11  # same host
192.168.1.12  # same host
192.168.1.13  # same host
```

## Node type logic

The playbook infers node type automatically:

### 1. Combined (host in both groups)
```yaml
# roles/linstor/controller/tasks/main.yaml:60-64
- name: initialize the LINSTOR control node as a Combined type
  shell: linstor node create ... --node-type Combined
  when: "'satellite' in group_names"
```

### 2. Controller (host only in controller)
```yaml
# roles/linstor/controller/tasks/main.yaml:54-58
- name: initialize the LINSTOR control node as pure Controller
  shell: linstor node create ... --node-type Controller
  when: "'satellite' not in group_names"
```

### 3. Satellite (host only in satellite)
```yaml
# roles/linstor/satellite/tasks/main.yaml:63-66
- name: join LINSTOR cluster as satellite node
  shell: linstor node create ... --node-type Satellite
```

## Example configuration

**This configuration is already Combined:**

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.11  # Combined
192.168.1.12  # Combined
192.168.1.13  # Combined
```

**Result:** All 3 hosts become **Combined** nodes.

## Pros and cons of Combined nodes

### Pros
1. **Resource saving** — no dedicated controller nodes
2. **Simplicity** — fewer nodes to manage
3. **Good for small clusters** — 3–5 nodes
4. **HA** — 3 Combined nodes provide HA for both controller and storage

### Cons
1. **Higher load per node** — two roles
2. **Less isolation** — storage issues can affect controller
3. **Less independent scaling** — harder to scale controller and storage separately

## When to use Combined

### Use Combined when:
- Small clusters (3–7 nodes)
- Test or dev
- Limited resources
- Simple deployments

### Prefer separate roles when:
- Large production (10+ nodes)
- Critical production
- Need independent scaling of controller and storage
- Strong isolation requirements

## Architecture with Combined nodes

```
┌─────────────────────────────────────────┐
│      LINSTOR Cluster (Combined)         │
├─────────────────────────────────────────┤
│                                         │
│  Combined Nodes (Controller + Satellite)│
│  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  │ .1.11    │  │ .1.12    │  │ .1.13    │
│  │ Combined │  │ Combined │  │ Combined │
│  │ Controller│  │ Controller│  │ Controller│
│  │ Satellite │  │ Satellite │  │ Satellite │
│  │ Storage   │  │ Storage   │  │ Storage   │
│  └──────────┘  └──────────┘  └──────────┘
│       │              │              │
│       └──────────────┴──────────────┘
│         DRBD Replication + HA
│                                         │
└─────────────────────────────────────────┘
```

## Storage pools for Combined nodes

### Option 1: No storage pool (file-thin only)
```ini
[satellite]
192.168.1.11
192.168.1.12
192.168.1.13

# No linstor_storage_pool group
```
→ Only file-thin pool on each node

### Option 2: With storage pool (lvm-thin)
```ini
[satellite]
192.168.1.11
192.168.1.12
192.168.1.13

[linstor_storage_pool]
192.168.1.11  # if disk /dev/sdb exists
192.168.1.12  # if disk /dev/sdb exists
192.168.1.13  # if disk /dev/sdb exists
```
→ lvm-thin pool on nodes that have the disk

**Important:** Ensure nodes have a free disk `/dev/sdb` (or the one set in `group_vars/all.yaml`).

## Post-install check

After running the playbook, check node types:

```bash
# On any node
linstor node list
```

You should see something like:
```
┌─────────────────────────────────────────┐
│ Node      │ Address    │ Type    │ ... │
├───────────┼────────────┼─────────┼─────┤
│ node-11   │ 192.168.1.11 │ Combined│ ... │
│ node-12   │ 192.168.1.12 │ Combined│ ... │
│ node-13   │ 192.168.1.13 │ Combined│ ... │
└───────────┴────────────┴─────────┴─────┘
```

## Summary

With the configuration above, all 3 nodes run as Combined, which fits a 3-node cluster well:
- HA (3 controllers)
- Data replication (3 satellites)
- Resource efficient
- Simple to operate
