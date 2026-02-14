# LINSTOR Ansible Playbook

Automated deployment of a LINSTOR® cluster with Ansible for Ubuntu 24.04+.

## Description

This playbook automatically configures and deploys a full LINSTOR cluster, including:

- System requirements check (Ubuntu 24.04+)
- DRBD and LVM installation and configuration
- LINSTOR Controller installation and configuration
- LINSTOR Satellite node installation and configuration
- Disk initialization and storage pool creation (lvm-thin)
- I/O scheduler tuning for DRBD disks
- LINSTOR GUI installation and access

## System requirements

- **OS**: Ubuntu 24.04 or newer (or compatible variants)
- **Ansible**: 2.9.0+ (latest recommended, e.g. 2.20.2+)
- **Python**: python-netaddr must be installed
- **SSH**: Passwordless SSH access to all target systems
- **Disks**: At least one extra disk for storage pool (defined in `[drbd]` section in inventory)
- **Network**: Separate network for DRBD replication (recommended but optional)

## Quick start

### 1. Inventory setup

Use an existing config file from the `clusters/` directory or create a new inventory file for your cluster.

**Example inventory structure:**

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.11
192.168.1.12
192.168.1.13

[linstor_storage_pool]
192.168.1.11
192.168.1.12
192.168.1.13

[admin]
192.168.1.11

[linstor_cluster:children]
controller
satellite

[drbd]
sdb
sdc
```

**Groups:**
- `[controller]` — LINSTOR controller nodes
- `[satellite]` — satellite nodes (can be Combined if also in controller group)
- `[linstor_storage_pool]` — nodes that provide storage pool
- `[admin]` — node for LINSTOR GUI installation
- `[drbd]` — list of disks for DRBD (sdb, sdc, etc.)

**Creating a new inventory file:**

Create a new file in `clusters/` (e.g. `clusters/my-cluster.ini`) and fill it following the example above with your nodes’ IPs and disks.

### 2. Variable configuration

Edit `group_vars/all.yaml`:

```yaml
---
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/ansible_key
ansible_become: yes

drbd_backing_disk: /dev/sdb
drbd_replication_network: 192.168.100.0/24
```

**Parameters:**
- `ansible_user` — SSH user
- `ansible_ssh_private_key_file` — path to SSH private key
- `drbd_backing_disk` — default disk for DRBD (if not set in [drbd] section)
- `drbd_replication_network` — DRBD replication network in CIDR format

### 3. Run installation

```bash
ansible-playbook -i clusters/<your-cluster>.ini ubuntu.yaml
```

Replace `<your-cluster>.ini` with your inventory filename (e.g. `clusters/linstor.ini`).

## Installation process

The playbook runs the following steps:

1. **Ansible temp directory** — creates `/tmp/ansible-tmp` on all nodes
2. **System check** — verifies Ubuntu version (24.04+ required)
3. **Disk check** — verifies disks from inventory `[drbd]` section
4. **Disk initialization** — creates PV, VG, thin pool and sets I/O scheduler
5. **Package installation** — installs DRBD, LVM and LINSTOR components
6. **Controller setup** — starts and configures LINSTOR Controller
7. **Satellite setup** — starts and configures LINSTOR Satellite nodes
8. **Node registration** — registers satellite nodes in the cluster
9. **Storage pool creation** — creates lvm-thin storage pools on nodes
10. **GUI installation** — installs LINSTOR GUI (bundled with controller)

See [docs/installation_process.md](docs/installation_process.md) for a detailed description.

## LINSTOR GUI access

After a successful run, the GUI is available in a browser at:

```
http://<controller-ip>:3370/ui/
```

**Default credentials:**
- Username: `admin`
- Password: `admin`

Change the password after first login.

## Verify installation

Connect to any controller and run:

```bash
# List nodes
linstor node list

# List storage pools
linstor storage-pool list

# List resources
linstor resource list
```

## Create a test resource

```bash
# Create resource definition
linstor resource-definition create test-res-0

# Create volume definition (100 MiB)
linstor volume-definition create test-res-0 100MiB

# Create resource on a node (use node name from linstor node list)
linstor resource create <node-name> test-res-0 --storage-pool lvm-thin

# Verify
linstor resource list
```

## Project structure

```
linstor-ansible/
├── ubuntu.yaml                  # Main playbook (Ubuntu)
├── clusters/                    # Inventory files
│   ├── linstor.ini             # Example cluster inventory
│   └── <your-cluster>.ini      # Your inventory file
├── group_vars/
│   └── all.yaml                # Global variables
├── docs/                        # Documentation
│   ├── installation_process.md # Installation process
│   ├── troubleshooting.md      # Troubleshooting
│   └── ...
└── roles/
    ├── commons/                 # Common roles
    │   ├── os-checker/         # OS and disk checks
    │   └── pre-install/        # Pre-install (packages, disks)
    └── linstor/                # LINSTOR roles
        ├── controller/         # LINSTOR controller
        ├── satellite/         # Satellite nodes
        ├── storage-pool/       # Storage pool creation
        └── gui/                # LINSTOR GUI
```

## Installed packages

The playbook installs the following packages from the LINBIT PPA:

- `drbd-utils` — DRBD utilities
- `drbd-dkms` — DRBD kernel module (DKMS)
- `lvm2` — Logical Volume Manager
- `linstor-controller` — LINSTOR controller
- `linstor-satellite` — LINSTOR satellite node
- `linstor-client` — LINSTOR client
- `linstor-gui` — Web UI (bundled with controller)

## Node types

### Controller
Node with LINSTOR controller only. Manages the cluster but does not store data.

### Satellite
Node with LINSTOR satellite only. Stores data but does not manage the cluster.

### Combined
Node with both controller and satellite. Can manage the cluster and store data.

To create a Combined node, add it to both `[controller]` and `[satellite]` in the inventory.

## Features

- ✅ **Ubuntu 24.04+ support**: Automatic OS version check
- ✅ **Automated installation**: Latest versions from LINBIT repos
- ✅ **Flexible configuration**: Multiple node types supported
- ✅ **Disk initialization**: Automatic PV, VG, thin pool creation
- ✅ **I/O tuning**: Scheduler configuration for DRBD disks
- ✅ **Built-in GUI**: Web UI available via controller

## Troubleshooting

See [docs/troubleshooting.md](docs/troubleshooting.md) for common issues.

### Quick checks

```bash
# Connection check
ansible all -i clusters/<your-cluster>.ini -m ping

# Ubuntu version
ansible all -i clusters/<your-cluster>.ini -m shell -a "lsb_release -a"

# Available disks
ansible linstor_storage_pool -i clusters/<your-cluster>.ini -m shell -a "lsblk -o NAME,SIZE,MODEL,SERIAL,ROTA,TYPE"
```

Replace `<your-cluster>.ini` with your inventory file name.

## Further documentation

- [Installation process](docs/installation_process.md) — detailed installation steps
- [Troubleshooting](docs/troubleshooting.md) — common problems and fixes
- [LINSTOR official documentation](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/)

## License

This project is distributed under the MIT license.

Copyright (c) 2026 Albert Iblyaminov

See [LICENSE](LICENSE) for details.

**Note:** LINSTOR is a LINBIT product and has its own license.
