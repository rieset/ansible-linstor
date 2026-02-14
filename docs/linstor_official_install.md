# Summary of official LINSTOR installation

Reference: [Official LINSTOR installation guide](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/#s-installation)

## Official installation steps

### 1. Register node in LINBIT Portal

**Manual:**
```bash
curl -O https://my.linbit.com/linbit-manage-node.py
chmod +x ./linbit-manage-node.py
./linbit-manage-node.py
```

**What the script does:**
- Detects OS and version (Ubuntu 22.04/24.04/25.04, RHEL 7/8/9)
- Registers the node in LINBIT Portal
- Configures LINBIT repositories for that OS
- Enables required repos (drbd-9, linstor, etc.)

**Ansible adaptation:** Fetch with `get_url`, run with `shell` and env vars; runs on all nodes.

### 2. Install packages

**RHEL/CentOS:**
```bash
yum install kmod-drbd drbd linstor-controller linstor-satellite linstor-client python-linstor
```

**Ubuntu/Debian:**
```bash
apt install drbd-dkms drbd-utils linstor-controller linstor-satellite linstor-client python-linstor
```

**Ansible:** `yum` and `apt` modules; no version pin (latest from repos); conditional on `ansible_os_family`.

### 3. Start services

**Controller:** `systemctl enable linstor-controller`; `systemctl start linstor-controller`  
**Satellite:** `systemctl enable linstor-satellite`; `systemctl start linstor-satellite`

**Ansible:** `systemd` module with `enabled: yes` and `state: restarted`.

### 4. Cluster initialization

**First node (Controller):**
```bash
linstor node create <node-name> <ip-address> --node-type Controller
```

**Satellite nodes:**
```bash
linstor node create <node-name> <ip-address> --node-type Satellite
```

**Combined:**
```bash
linstor node create <node-name> <ip-address> --node-type Combined
```

**Ansible:** Done via `shell`; node type comes from inventory groups; `ipaddr` filter for replication network IP.

## How this playbook differs from manual install

- All steps run on all nodes automatically; single config for the cluster.
- Variables in `group_vars/all.yaml`; credentials can be passed on the command line; inventory for topology.
- Firewall and LINSTOR client config via templates; same setup on every node.
- File-thin pool on all satellites; lvm-thin on nodes with disks; LVM VG and thin pools created by the playbook.

## Mapping to official steps

| Official step       | Playbook                         | File / role                    |
|---------------------|----------------------------------|--------------------------------|
| Node registration   | linbit-manage-node.py            | pre-install/tasks/pkg.yaml     |
| Install packages    | yum/apt                          | pre-install/tasks/pkg.yaml     |
| Start Controller    | systemd                           | linstor/controller/tasks       |
| Start Satellite     | systemd                           | linstor/satellite/tasks        |
| Node init           | linstor node create              | controller + satellite tasks   |
| Firewall            | firewalld/ufw                     | linstor firewall role          |
| Storage pools       | linstor storage-pool create      | satellite + storage-pool      |

## Supported OS

- **RHEL:** 7, 8, 9  
- **Ubuntu:** 22.04, 24.04, 25.04+  
- **CentOS:** via RedHat family

## Requirements

- LINBIT Portal account (https://my.linbit.com)
- Contract ID and Cluster ID from the portal
- SSH access to all nodes
- Python on the control machine (for Ansible)

## Links

- [LINSTOR official documentation](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/)
- [LINBIT Portal](https://my.linbit.com)
- [LINSTOR introduction](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/#p-linstor-introduction)
