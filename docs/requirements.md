Выполняем эти операции на каждой ноде:


1) Смотрим диски

```yaml
lsblk -o NAME,SIZE,MODEL,SERIAL,ROTA,TYPE
```


Тома sda…sdh должны корректно отображаться в системе.


2) Создаем VG:

```yaml
pvcreate /dev/sd{a,b,c,d,e,f,g,h}
vgcreate drbdpool /dev/sd{a,b,c,d,e,f,g,h}
```


3) Создаем thin pool:

```javascript
lvcreate -l 95%VG -T drbdpool/thinpool
```


4) Настраиваем I/O scheduler и queue:

```javascript
for d in sda sdb sdc sdd sde sdf sdg sdh; do
  echo mq-deadline | sudo tee /sys/block/$d/queue/scheduler
  echo 0 | sudo tee /sys/block/$d/queue/add_random
  echo 1 | sudo tee /sys/block/$d/queue/nomerges
done
```


Официальный способ (одна команда):

```javascript
kubectl apply --server-side -f \
  "https://github.com/piraeusdatastore/piraeus-operator/releases/latest/download/manifest.yaml"
```


Ждём, пока он поднимется:

```javascript
kubectl wait pod \
  --for=condition=Ready \
  -n piraeus-datastore \
  -l app.kubernetes.io/component=piraeus-operator
```


Проверка:

```javascript
kubectl -n piraeus-datastore get pods
```


## Создаем LINSTOR Cluster (CR LinstorCluster)


Минимальный CR – оператор сам поднимет controller, satellites, DRBD, CSI, HA-controller:

```javascript
kubectl apply -f - <<'EOF'
apiVersion: piraeus.io/v1
kind: LinstorCluster
metadata:
  name: linstorcluster
spec: {}
EOF
```


Ждем, пока кластер не будет готов:

```javascript
kubectl get linstorcluster linstorcluster -o yaml \
  | yq '.status.conditions'
```

Статус должен быть Available/Configured/Applied=True в conditions.


Или на коленке:

```javascript
kubectl get linstorcluster linstorcluster
```


Смотрим поды:

```javascript
kubectl -n piraeus-datastore get pods
```


Должно быть:

* linstor-controller-...
* linstor-satellite-... DaemonSet’ы (по одному pod на каждый node)
* csi-controller-..., csi-node-...
* ha-controller-...


## Включаем hostNetwork для Satellite (DRBD по IP ноды)


Чтобы DRBD не зависел от kube-ovn overlay, включаем hostNetwork для satellite’ов (официальный пример):

```javascript
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


После применения (заменить NODE_NAME и выполнить для каждой моды):

```javascript
kubectl -n piraeus-datastore rollout restart daemonset linstor-satellite.{NODE_NAME}
kubectl -n piraeus-datastore get pods -o wide
```


Все Satellite pod’ы будут иметь IP хоста. DRBD-репликация пойдёт через host network, что нам и нужно.

