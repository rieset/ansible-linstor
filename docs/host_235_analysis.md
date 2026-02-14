# Host without disk for storage pool (example)

Example of handling a satellite node that has no dedicated disk for `drbd_backing_disk`.

## Check results

### Host is reachable
- SSH works
- User: `<ansible_user>`
- Can run commands with sudo

### Problem: no disk for drbd_backing_disk

**Situation:**
- Host has only **one disk**: `/dev/sda` (33.5G)
- `/dev/sda` is fully used by the system:
  - `sda1` (32.5G) — root `/`
  - `sda14` (4M) — partition
  - `sda15` (106M) — `/boot/efi`
  - `sda16` (913M) — `/boot`
- **No `/dev/sdb`** as set in `group_vars/all.yaml`
- No LVM volume groups

**Example inventory:**
```ini
[satellite]
192.168.1.25  # present

[linstor_storage_pool]
192.168.1.25  # problem: expects /dev/sdb which is missing
```

## Effect

With this configuration:
- Role `linstor/satellite` will succeed (creates file-thin pool)
- Role `linstor/storage-pool` will **fail** because `/dev/sdb` does not exist

## Options

### Option 1: Remove host from linstor_storage_pool (recommended)

**If you cannot add a disk**, remove the host from `linstor_storage_pool`:

```ini
[satellite]
192.168.1.25
192.168.1.26
192.168.1.27

[linstor_storage_pool]
192.168.1.26  # omit 192.168.1.25
192.168.1.27
```

**Result:**
- Host works as satellite
- File-thin storage pool is created (no disk required)
- Lower performance than lvm-thin pool

### Option 2: Add a disk to the server

If you can add a disk:

1. Attach a disk to the VM/server
2. Check that the disk appears:
   ```bash
   ssh <user>@<host> "sudo lsblk"
   ```
3. If the disk appears (e.g. `/dev/sdb`):
   - Keep the current config
   - The playbook will create the LVM thin pool on it

### Option 3: Use another existing disk (not recommended)

**Warning:** This host has no other free disks; this option does not apply here.

If there were another free disk (e.g. `/dev/sdc`), you could:
1. Change `group_vars/all.yaml`:
   ```yaml
   drbd_backing_disk: /dev/sdc
   ```
2. Or use host-level variables

## Recommendation

**For a host without a disk, use Option 1:**

1. Remove the host from `[linstor_storage_pool]`
2. Keep it only in `[satellite]`
3. The host will use file-thin pool (enough for many workloads)

**Or**, for higher performance:
- Add a disk to the server
- Keep the host in `[linstor_storage_pool]`

## Example cluster layout

After applying Option 1:

```
Controllers: 3 nodes (192.168.1.11–13)
Satellites: 3 nodes (192.168.1.25–27)
Storage Pools:
  - lvm-thin: 2 nodes (192.168.1.26–27) — higher performance
  - file-thin: 1 node (192.168.1.25) — standard
```

This is a valid mixed setup.

## Verification commands

After changing the config:

```bash
# Connectivity
ansible 192.168.1.25 -i clusters/<your-cluster>.ini -m ping

# Syntax check
ansible-playbook ubuntu.yaml --syntax-check

# Dry run for this host only
ansible-playbook ubuntu.yaml -i clusters/<your-cluster>.ini --limit 192.168.1.25 --check
```
