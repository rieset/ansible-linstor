# LINSTOR installation process

Detailed description of the steps to install a LINSTOR cluster with the Ansible playbook.

## Overview

Playbook `ubuntu.yaml` runs the installation in this order:

1. Environment preparation
2. System check
3. Disk initialization
4. Package installation
5. Controller setup
6. Satellite setup
7. Node registration
8. Storage pool creation
9. GUI installation

## Stage 1: Environment preparation

**Play:** `linstor_cluster`  
**File:** `ubuntu.yaml` (lines 2–10)

### Tasks

- Create temporary directory `/tmp/ansible-tmp` with mode 1777 on all cluster nodes
- This directory is used by Ansible for temporary files during task execution

### Output

- Directory `/tmp/ansible-tmp` exists on all nodes

---

## Stage 2: System check

**Play:** `linstor_cluster` (via role `commons/os-checker`)  
**File:** `roles/commons/os-checker/tasks/main.yaml`

### Tasks

1. **OS check**
   - Detect OS type from `/etc/os-release`
   - Ensure it is Ubuntu
   - Fail if OS is not Ubuntu

2. **Ubuntu version check**
   - Read version from `/etc/os-release`
   - Ensure version >= 24.04
   - Fail if version < 24.04

3. **Disk check**
   - Get disk list from inventory `[drbd]` section
   - Check presence of each disk (`/dev/sdb`, `/dev/sdc`, etc.)
   - Print list of available disks for debugging
   - Fail if disks are not found

### Output

- Variable `found_disks` contains the list of found disks
- Error if system does not meet requirements

### Error handling

If disks are not found, the playbook prints detailed instructions:
- How to check available disks (`lsblk`)
- How to add a disk to a VM
- How to update inventory or variables

---

## Stage 3: Disk initialization

**Play:** `linstor_storage_pool`  
**File:** `ubuntu.yaml` (lines 14–20), `roles/linstor/storage-pool/tasks/disks.yaml`

### Tasks

1. **Create Physical Volumes (PV)**
   - Check for existing PV on each disk
   - Create PV on disks without PV (`pvcreate`)

2. **Create Volume Group (VG)**
   - Check for existing VG `drbdpool`
   - Create VG `drbdpool` with all found disks (`vgcreate`)
   - Extend existing VG with new disks (`vgextend`)

3. **Create thin pool**
   - Check for existing thin pool `drbdpool/thinpool`
   - Create thin pool at 95% of VG size (`lvcreate -l 95%VG -T`)

4. **I/O scheduler**
   - Set scheduler `mq-deadline` for each disk
   - Disable `add_random` (set to 0)
   - Enable `nomerges` (set to 1)

### Output

- PVs created on all disks
- VG `drbdpool` created or extended
- Thin pool `drbdpool/thinpool` created (95% of VG)
- I/O scheduler configured for all disks

### Note

This stage runs **before** LINSTOR nodes are created, as these are local operations and do not require cluster connectivity.

---

## Stage 4: Package installation

**Play:** `controller` and `satellite` (via role `commons/pre-install`)  
**File:** `roles/commons/pre-install/tasks/pkg.yaml`

### Tasks

1. **Clean conflicting configs**
   - Find existing LINBIT PPA files in `/etc/apt/sources.list.d/`
   - Find files mentioning `linbit` by content
   - Remove found conflicting configs
   - Remove old LINBIT GPG keys

2. **Install dependencies**
   - Install kernel headers (`linux-headers-$(uname -r)`)
   - Install `software-properties-common` (for `add-apt-repository`)

3. **Repository setup**
   - Add LINBIT PPA (`ppa:linbit/linbit-drbd9-stack`)
   - Update apt cache

4. **Install packages**
   - `drbd-utils` — DRBD utilities
   - `drbd-dkms` — DRBD kernel module (DKMS)
   - `lvm2` — Logical Volume Manager
   - Packages from variable `lb_deb_pkgs` (if defined)

5. **Load DRBD module**
   - Load `drbd` kernel module (`modprobe drbd`)
   - Enable load at boot (`/etc/modules-load.d/drbd.conf`)

### Output

- All required packages installed
- DRBD module loaded and configured for autoload
- LINBIT repository configured

---

## Stage 5: Controller setup

**Play:** `controller`  
**File:** `roles/linstor/controller/tasks/main.yaml`

### Tasks

1. **Check and start service**
   - Check that service `linstor-controller` exists
   - Enable and start service (`systemctl enable --now`)
   - Wait for service readiness (port 3370)

2. **Register satellite nodes**
   - Identify first controller
   - For each host in group `[satellite]`:
     - Get node name (`ansible_nodename` or IP)
     - Get IP from `drbd_replication_network`
     - Determine node type (Combined if host is also in controller group)
     - Create node in LINSTOR (`linstor node create`)
   - Wait for satellites to connect

3. **Status check**
   - Check node status via `linstor node list`
   - Print registration results

### Output

- Service `linstor-controller` started and enabled
- All satellite nodes registered in the cluster
- Nodes connected to controller

### Node type logic

- **Combined**: host is in both `[controller]` and `[satellite]`
- **Satellite**: host is only in `[satellite]`
- **Controller**: host is only in `[controller]` (not registered as satellite)

---

## Stage 6: Satellite setup

**Play:** `satellite`  
**File:** `roles/linstor/satellite/tasks/main.yaml`

### Tasks

1. **Check and start service**
   - Check that service `linstor-satellite` exists
   - Enable and start service (`systemctl enable --now`)

2. **Configuration**
   - Create directory `/etc/linstor` if missing
   - Create `/etc/linstor/linstor-client.conf` with all controller IPs
   - Restart service after config update

### Config format

```ini
[global]
controllers=192.168.1.11,192.168.1.12,192.168.1.13
```

### Output

- Service `linstor-satellite` started and enabled
- Config `/etc/linstor/linstor-client.conf` created
- Service restarted with new config

---

## Stage 7: Node registration

**Play:** `controller` (part of stage 5)  
**File:** `roles/linstor/controller/tasks/main.yaml`

This stage runs as part of Controller setup (stage 5) but is listed separately for clarity.

### Process

1. Controller is up and accepting connections
2. Satellite nodes are up and configured
3. Controller registers each satellite with:
   ```bash
   linstor node create <node-name> <ip-address> --node-type <type>
   ```
4. Satellites connect to the controller automatically
5. Wait for all nodes to connect (60s timeout)

### Output

- All nodes registered in the cluster
- All nodes connected to the controller
- Node status: `ONLINE`

---

## Stage 8: Storage pool creation

**Play:** `linstor_storage_pool`  
**File:** `ubuntu.yaml` (lines 37–74), `roles/linstor/storage-pool/tasks/main.yaml`

### Tasks

1. **Wait for readiness**
   - Wait for controller (port 3370)
   - Wait for all satellites to connect

2. **Check disk initialization**
   - Check that thin pool `drbdpool/thinpool` exists
   - Fail if thin pool not found

3. **Create storage pool**
   - Get node name (`ansible_nodename` or IP)
   - Create lvm-thin storage pool from first controller:
     ```bash
     linstor storage-pool create lvmthin <node-name> lvm-thin drbdpool/thinpool
     ```
   - Print creation result

### Output

- Storage pool `lvm-thin` created on each host in `linstor_storage_pool`
- Pool uses thin pool `drbdpool/thinpool`

### Error handling

- Exit code 0: created successfully
- Exit code 10: pool already exists (not an error)
- Other codes: creation failed

---

## Stage 9: GUI installation

**Play:** `admin`  
**File:** `roles/linstor/gui/tasks/main.yaml`

### Tasks

1. **Wait for controllers**
   - Wait for all controllers (port 3370)

2. **Attempt package install**
   - Update apt cache
   - Check availability of package `linstor-gui`
   - Install if available
   - Check that service `linstor-gui` exists

3. **Run via systemd (if package installed)**
   - Enable and start service `linstor-gui`

4. **Docker fallback (if package unavailable)**
   - Check for Docker container
   - Install Docker and Docker Compose if missing
   - Create directory `/opt/linstor-gui`
   - Create `docker-compose.yml` with image `linbit/linstor-gui:latest`
   - Start container with `docker-compose` or `docker compose`

### Note

**LINSTOR GUI is built into the controller** and is available on port 3370 at `/ui/`. The `linstor-gui` package only contains static web UI files.

### GUI access

After installation, GUI is available at:
```
http://<controller-ip>:3370/ui/
```

**Default credentials:**
- Username: `admin`
- Password: `admin`

### Output

- GUI installed and available via controller
- URL: `http://<controller-ip>:3370/ui/`

---

## Execution order

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Environment preparation (linstor_cluster)               │
│    └─> Create /tmp/ansible-tmp                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. System check (linstor_cluster)                           │
│    └─> Check Ubuntu 24.04+, check disks                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Disk initialization (linstor_storage_pool)                 │
│    └─> PV, VG, thin pool, I/O scheduler                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Package installation (controller + satellite)            │
│    └─> DRBD, LVM, LINSTOR components                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
        ┌───────────────────┴───────────────────┐
        ↓                                       ↓
┌──────────────────────┐          ┌──────────────────────┐
│ 5. Controller setup  │          │ 6. Satellite setup   │
│    └─> Start service │          │    └─> Start service │
└──────────────────────┘          └──────────────────────┘
        ↓                                       ↓
        └───────────────────┬───────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. Node registration (controller)                            │
│    └─> linstor node create for each satellite               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. Storage pool creation (linstor_storage_pool)             │
│    └─> linstor storage-pool create lvmthin                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 9. GUI installation (admin)                                 │
│    └─> Package or Docker container                          │
└─────────────────────────────────────────────────────────────┘
```

## Runtime

Approximate runtime for a 3-node cluster:

- Environment preparation: ~5 s
- System check: ~10 s
- Disk initialization: ~30 s (depends on disk size)
- Package installation: ~2–5 min (depends on network)
- Controller/Satellite setup: ~30 s
- Node registration: ~1 min
- Storage pool creation: ~10 s
- GUI installation: ~30 s

**Total:** ~5–10 minutes for a typical cluster.

## Post-install checks

After installation:

```bash
# On any controller
linstor node list
linstor storage-pool list
linstor resource list

# Service status
systemctl status linstor-controller
systemctl status linstor-satellite

# GUI check
curl -I http://<controller-ip>:3370/ui/
```

## Rollback

The playbook is idempotent and can be run multiple times. For a full rollback:

1. Stop services:
   ```bash
   systemctl stop linstor-controller linstor-satellite
   ```

2. Remove nodes from cluster (on controller):
   ```bash
   linstor node delete <node-name>
   ```

3. Delete storage pools (on controller):
   ```bash
   linstor storage-pool delete <node-name> <pool-name>
   ```

4. Remove LVM structures (on each node):
   ```bash
   lvremove drbdpool/thinpool
   vgremove drbdpool
   pvremove /dev/sdb /dev/sdc
   ```

5. Remove packages:
   ```bash
   apt remove linstor-* drbd-* lvm2
   ```
