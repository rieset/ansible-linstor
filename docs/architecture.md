# LINSTOR architecture and configuration overview

## Component roles

### 1. Controller

**Role:** Central management for the LINSTOR cluster

**Functions:**
- Cluster metadata (resource definitions, volume definitions)
- Coordination between satellite nodes
- REST API (port 3370) for cluster management
- Replication and data placement
- State of all nodes and resources

**Details:**
- Service: `linstor-controller.service`
- Ports: 3370 (REST API), 3376 (satellite communication)
- Node type: `Controller` (controller only) or `Combined` (controller + satellite)

**When to use:**
- At least 1 controller (3 recommended for HA)
- Controllers do not store data, only manage the cluster
- Odd number (1, 3, 5) recommended for HA

### 2. Satellite

**Role:** Nodes that store and replicate data

**Functions:**
- Store data on local disks
- Replicate data via DRBD to other satellites
- Execute controller commands
- Manage local storage pools
- Expose block devices to applications

**Details:**
- Service: `linstor-satellite.service`
- Ports: 3366 (controller), 7000–8000 (DRBD replication)
- Node type: `Satellite` (satellite only) or `Combined` (controller + satellite)
- Automatically creates file-thin storage pool

**When to use:**
- All nodes that should store data
- At least 2 satellites for replication
- Can run as Combined (controller + satellite)

### 3. Storage pool

**Role:** Defines which nodes provide block storage

**Functions:**
- Create LVM volume group `drbdpool` on the given disk
- Create LVM thin pool `thinpool` (e.g. 50% of VG)
- Register lvm-thin storage pool in LINSTOR
- Provide higher-performance block storage

**Details:**
- Requires: unused block disk (`drbd_backing_disk`)
- Creates: VG `drbdpool` → thin pool `thinpool` → LINSTOR pool `lvm-thin`
- If no disk: only file-thin pool (lower performance)

**When to use:**
- Nodes with dedicated storage disks
- Production (lvm-thin is faster than file-thin)
- Not required on every satellite

## Node types in LINSTOR

### Controller (controller only)
- Management only, no data storage
- Created when host is in `controller` but NOT in `satellite`

### Satellite (satellite only)
- Data storage only, does not manage cluster
- Created when host is in `satellite` but NOT in `controller`

### Combined
- Both controller and satellite
- Created when host is in both `controller` and `satellite`
- Useful for small clusters (saves resources)

## Example configuration

### Example inventory:

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.21
192.168.1.22
192.168.1.23

[linstor_cluster:children]
controller
satellite

[linstor_storage_pool]
192.168.1.21
192.168.1.22
192.168.1.23
```

### Pros

1. **Role separation:** 3 controllers separate from satellites; clear split between management and storage.
2. **Controller HA:** 3 controllers give HA (odd number for quorum); cluster keeps running if one fails.
3. **Storage scale:** 3 satellites for replication; 2–3 replicas possible.
4. **Storage pools:** All satellites provide block storage; lvm-thin pool on each node.

### Considerations

1. **No Combined nodes:** One controller failure is tolerated; for very small clusters, Combined can save resources.
2. **All satellites in storage pool:** Ensure each satellite has a free disk (`drbd_backing_disk`); remove from `linstor_storage_pool` if not.
3. **Replication network:** Set `drbd_replication_network` correctly; use a dedicated network (not management).
4. **Replication:** With 3 satellites you can use 2 or 3 replicas; 3 replicas need at least 3 satellites.

### Architecture diagram

```
┌─────────────────────────────────────────────────────────┐
│                    LINSTOR Cluster                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Controllers (Management Layer)                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ .1.11    │  │ .1.12    │  │ .1.13    │               │
│  │Controller│  │Controller│  │Controller│               │
│  └──────────┘  └──────────┘  └──────────┘               │
│       │              │              │                    │
│       └──────────────┴──────────────┘                    │
│                    │                                     │
│                    ▼                                     │
│  Satellites (Storage Layer)                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ .1.21    │  │ .1.22    │  │ .1.23    │               │
│  │ Satellite│  │ Satellite│  │ Satellite│                │
│  │ +Storage │  │ +Storage │  │ +Storage │                │
│  │  Pool    │  │  Pool    │  │  Pool    │                │
│  └──────────┘  └──────────┘  └──────────┘                │
│       │              │              │                    │
│       └──────────────┴──────────────┘                    │
│              DRBD Replication                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Recommendations

1. **Production:** This layout is good; consider one more satellite for resilience.
2. **Test/dev:** Combined nodes (controller + satellite on one host) are fine; minimum: 1 Combined or 1 Controller + 2 Satellites.
3. **Monitoring:** Monitor all nodes; controllers are critical.
4. **Backup:** Back up controller metadata; use replication between satellites.

### Optional: Combined example

To use Combined nodes and save resources:

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.11  # Combined: controller + satellite
192.168.1.12  # Combined: controller + satellite
192.168.1.13  # Combined: controller + satellite
192.168.1.21  # Pure satellite
192.168.1.22  # Pure satellite
192.168.1.23  # Pure satellite

[linstor_cluster:children]
controller
satellite

[linstor_storage_pool]
192.168.1.11
192.168.1.12
192.168.1.13
192.168.1.21
192.168.1.22
192.168.1.23
```

This configuration is suitable for production with high availability and scalability.
