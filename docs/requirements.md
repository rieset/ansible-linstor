# Requirements and manual steps

Operations to run on each node (or reference for automation):

## 1. Check disks

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,ROTA,TYPE
```

Volumes sda…sdh should be visible.

## 2. Create VG

```bash
pvcreate /dev/sd{a,b,c,d,e,f,g,h}
vgcreate drbdpool /dev/sd{a,b,c,d,e,f,g,h}
```

## 3. Create thin pool

```bash
lvcreate -l 95%VG -T drbdpool/thinpool
```

## 4. I/O scheduler and queue

```bash
for d in sda sdb sdc sdd sde sdf sdg sdh; do
  echo mq-deadline | sudo tee /sys/block/$d/queue/scheduler
  echo 0 | sudo tee /sys/block/$d/queue/add_random
  echo 1 | sudo tee /sys/block/$d/queue/nomerges
done
```

## Official Kubernetes operator (one command)

```bash
kubectl apply --server-side -f \
  "https://github.com/piraeusdatastore/piraeus-operator/releases/latest/download/manifest.yaml"
```

Wait for the operator:

```bash
kubectl wait pod \
  --for=condition=Ready \
  -n piraeus-datastore \
  -l app.kubernetes.io/component=piraeus-operator
```

Check:

```bash
kubectl -n piraeus-datastore get pods
```

## Create LINSTOR cluster (LinstorCluster CR)

Minimal CR; operator brings up controller, satellites, DRBD, CSI, HA-controller:

```bash
kubectl apply -f - <<'EOF'
apiVersion: piraeus.io/v1
kind: LinstorCluster
metadata:
  name: linstorcluster
spec: {}
EOF
```

Wait until the cluster is ready:

```bash
kubectl get linstorcluster linstorcluster -o yaml | yq '.status.conditions'
```

Status conditions should show Available/Configured/Applied=True.

Or:

```bash
kubectl get linstorcluster linstorcluster
```

Check pods:

```bash
kubectl -n piraeus-datastore get pods
```

Expected: linstor-controller-..., linstor-satellite-... (DaemonSet), csi-controller-..., csi-node-..., ha-controller-...

## Enable hostNetwork for Satellite (DRBD on node IP)

So DRBD does not depend on overlay; enable hostNetwork for satellites (official example):

```bash
kubectl apply -f - <<'EOF'
apiVersion: piraeus.io/v1
kind: LinstorSatelliteConfiguration
metadata:
  name: host-network
spec:
  podTemplate:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
EOF
```

After applying (replace NODE_NAME and run per node):

```bash
kubectl -n piraeus-datastore rollout restart daemonset linstor-satellite.{NODE_NAME}
kubectl -n piraeus-datastore get pods -o wide
```

Satellite pods will use the host IP; DRBD replication will use the host network.
