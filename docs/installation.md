# LINSTOR installation guide

## Prerequisites

1. **Ansible** 2.9.0 or newer (latest recommended)
   ```bash
   ansible --version
   ```
   Minimum version is 2.9.0+; latest stable (e.g. 2.20.2+) is recommended.

2. **Python netaddr library**
   ```bash
   pip install netaddr
   ```

3. **SSH access** to all target hosts without password (key-based)

4. **LINBIT Portal account** at https://my.linbit.com

## Step 1: Inventory setup

Use a file from `clusters/` (e.g. `clusters/linstor.ini`) or create your own. Set your servers’ IP addresses or hostnames:

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.21
192.168.1.22
192.168.1.23

[linstor_storage_pool]
192.168.1.21
192.168.1.22
192.168.1.23

[admin]
192.168.1.11

[linstor_cluster:children]
controller
satellite

[drbd]
sdb
sdc
```

**Notes:**
- A host can be in both `controller` and `satellite` — that creates a Combined node
- Hosts in `linstor_storage_pool` provide block storage (LVM thin pool)
- `[admin]` — host where LINSTOR GUI is installed (when using the gui role)
- `[drbd]` — list of disks for storage pool (sdb, sdc, etc.)

## Step 2: Variable configuration

Edit `group_vars/all.yaml`:

```yaml
---
# Ansible variables
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/ansible_key
ansible_become: yes

# LINSTOR variables
drbd_backing_disk: /dev/sdb
drbd_replication_network: 192.168.100.0/24

# LINBIT portal variables
lb_user: ""
lb_pass: ""
lb_con_id: ""
lb_clu_id: ""
```

### Parameters

- **ansible_user** — SSH user
- **ansible_ssh_private_key_file** — path to SSH private key
- **ansible_become** — privilege escalation (playbook uses `become: true`)
- **drbd_backing_disk** — unused block device for LVM (e.g. `/dev/sdb`)
  - If no such disk, do not add the host to `linstor_storage_pool` — only file-thin pool will be used
- **drbd_replication_network** — DRBD replication network in CIDR (separate network recommended)
- **lb_user, lb_pass** — LINBIT Portal credentials
- **lb_con_id** — Contract ID in LINBIT Portal
- **lb_clu_id** — Cluster ID in LINBIT Portal

## Step 3: Connection check

Ensure Ansible can reach all hosts:

```bash
ansible all -i clusters/linstor.ini -m ping
```

You should see `SUCCESS` for all hosts. Replace `linstor.ini` with your inventory file if needed.

## Step 4: Run installation

### Basic run (uses variables from group_vars/all.yaml)

```bash
ansible-playbook ubuntu.yaml
```

### With explicit inventory

```bash
ansible-playbook -i clusters/linstor.ini ubuntu.yaml
```

### Passing LINBIT credentials on the command line

If you prefer not to store passwords in a file:

```bash
ansible-playbook ubuntu.yaml \
  -e lb_user="your_login" \
  -e lb_pass="your_password" \
  -e lb_con_id="1234" \
  -e lb_clu_id="4321"
```

### Limiting to specific hosts

```bash
# Controller only
ansible-playbook ubuntu.yaml --limit controller

# Satellite only
ansible-playbook ubuntu.yaml --limit satellite

# Single host
ansible-playbook ubuntu.yaml --limit 192.168.1.11
```

### Running by tags (specific roles)

```bash
# Controller only
ansible-playbook ubuntu.yaml --tags controller

# Satellite only
ansible-playbook ubuntu.yaml --tags satellite

# Storage pool only
ansible-playbook ubuntu.yaml --tags storage-pool
```

### Dry run

```bash
ansible-playbook ubuntu.yaml --check
```

### Verbose output

```bash
ansible-playbook ubuntu.yaml -v    # -v, -vv, -vvv for more detail
```

## Step 5: Verify installation

After a successful run, connect to a controller and run:

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

# Create volume definition
linstor volume-definition create test-res-0 100MiB

# Create resource on a node
linstor resource create \
  $(linstor sp list | head -n4 | tail -n1 | cut -d"|" -f3 | sed 's/ //g') \
  test-res-0 --storage-pool lvm-thin

# Verify
linstor resource list
```

## Troubleshooting

### SSH connection errors

```bash
# Check SSH key
ssh -i ~/.ssh/ansible_key ansible@192.168.1.11

# Ensure key is in ssh-agent
ssh-add ~/.ssh/ansible_key
```

### LINBIT Portal registration errors

- Check credentials
- Ensure you have an active contract
- Check https://my.linbit.com is reachable

### Package installation errors

- Ensure nodes have internet access
- Check LINBIT repos are configured correctly after registration
- Check OS version is supported

### Firewall issues

The playbook configures the firewall automatically; if you have issues:

```bash
# RHEL/CentOS
firewall-cmd --list-ports

# Ubuntu
ufw status
```

## Useful commands

### Ansible configuration

```bash
ansible-config dump
```

### Playbook syntax check

```bash
ansible-playbook ubuntu.yaml --syntax-check
```

### Host facts

```bash
ansible all -i clusters/linstor.ini -m setup
```

### Run command on all hosts

```bash
ansible all -i clusters/linstor.ini -a "systemctl status linstor-controller"
```

## Usage examples

### Install only on new nodes

```bash
# Add new nodes to inventory, then:
ansible-playbook ubuntu.yaml --limit new_nodes
```

### Update controller only

```bash
ansible-playbook ubuntu.yaml --tags controller --limit controller
```

### Recreate storage pool

```bash
ansible-playbook ubuntu.yaml --tags storage-pool
```
