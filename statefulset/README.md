# Kubernetes StatefulSet — Hands-on Session

## What is a StatefulSet?

A **StatefulSet** manages pods that need stable, persistent identity. Unlike a Deployment (where pods are interchangeable), each StatefulSet pod gets:

- A **stable, ordered name** — `nginx-sts-0`, `nginx-sts-1`, `nginx-sts-2` (never random hashes).
- A **stable network identity** via a headless service — `nginx-sts-2.nginx-headless` is always that exact pod.
- A **dedicated PersistentVolumeClaim** — each pod owns its own volume; the volume survives pod restarts and rescheduling.

Pods are created and deleted **in order**: 0 → 1 → 2 on scale-up, 2 → 1 → 0 on scale-down.

Typical real-world uses: databases (MySQL, PostgreSQL, MongoDB), message brokers (Kafka, RabbitMQ), distributed stores (Zookeeper, etcd).

---

## Object hierarchy

```
StatefulSet (nginx-sts)
  ├── Headless Service (nginx-headless)   ← provides stable DNS
  ├── Pod nginx-sts-0  +  PVC www-nginx-sts-0
  ├── Pod nginx-sts-1  +  PVC www-nginx-sts-1
  └── Pod nginx-sts-2  +  PVC www-nginx-sts-2
```

---

## Manifest — `statefulset.yaml`

Two resources are defined in one file, separated by `---`.

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None          # <-- this is what makes it "headless"
  ports:
  - port: 80
    name: web
  selector:
    run: nginx-sts-demo
```

| Field | Value | Meaning |
|---|---|---|
| `clusterIP: None` | None | No virtual IP; DNS returns individual pod IPs directly |
| `selector` | `run=nginx-sts-demo` | Routes to pods with this label |

A normal Service gives you one stable IP that load-balances across pods. A **headless** Service skips the VIP and instead lets DNS resolve to each pod individually — this is what makes `nginx-sts-2.nginx-headless` work.

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-sts
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      run: nginx-sts-demo
  template:
    metadata:
      labels:
        run: nginx-sts-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: www
          mountPath: /var/www/
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      storageClassName: hostpath
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Mi
```

| Field | Value | Meaning |
|---|---|---|
| `serviceName` | `nginx-headless` | Links StatefulSet to the headless service for DNS |
| `replicas` | 3 | Create 3 pods (0, 1, 2) |
| `volumeMounts.mountPath` | `/var/www/` | Where the PVC is mounted inside the container |
| `volumeClaimTemplates` | — | Blueprint used to create one PVC **per pod** automatically |
| `storageClassName` | `hostpath` | Docker Desktop's built-in storage class |
| `accessModes` | `ReadWriteOnce` | Volume can be mounted read-write by one node at a time |
| `storage` | `10Mi` | Size of each pod's volume |

**`volumeClaimTemplates`** is the key StatefulSet-only feature. Kubernetes stamps out a unique PVC for each pod using the template: `www-nginx-sts-0`, `www-nginx-sts-1`, `www-nginx-sts-2`.

---

## Commands Executed

### 1. Apply the manifest
```powershell
kubectl apply -f statefulset.yaml
# service/nginx-headless created
# statefulset.apps/nginx-sts created
```
Both the headless Service and the StatefulSet are created. Pods are started in order: `-0` first, then `-1`, then `-2`.

---

### 2. List pods (wide output)
```powershell
kubectl get pods -o wide
```
```
NAME          READY   STATUS    RESTARTS   AGE   IP          NODE
nginx-sts-0   1/1     Running   0          18s   10.1.0.26   docker-desktop
nginx-sts-1   1/1     Running   0          13s   10.1.0.27   docker-desktop
nginx-sts-2   1/1     Running   0           7s   10.1.0.28   docker-desktop
```
- Ordered names (`-0`, `-1`, `-2`) — not random hashes.
- Each pod got its own cluster IP.
- The 5-second gaps between ages reflect the ordered startup.

---

### 3. List PersistentVolumeClaims
```powershell
kubectl get pvc
```
```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
www-nginx-sts-0   Bound    pvc-42a9ca69-...                           10Mi       RWO            hostpath
www-nginx-sts-1   Bound    pvc-2665a00f-...                           10Mi       RWO            hostpath
www-nginx-sts-2   Bound    pvc-fcd1ff20-...                           10Mi       RWO            hostpath
```
One PVC per pod, named `www-<pod-name>`. All `Bound` — each is backed by a real PersistentVolume on the host.

---

### 4. Describe pod `nginx-sts-2`
```powershell
kubectl describe po nginx-sts-2
```
Key observations:

| Detail | Value |
|---|---|
| Labels | `apps.kubernetes.io/pod-index=2`, `statefulset.kubernetes.io/pod-name=nginx-sts-2` |
| Controlled By | `StatefulSet/nginx-sts` |
| Volume mount | `/var/www/` ← `www-nginx-sts-2` (PVC) |
| QoS Class | `BestEffort` (no resource requests/limits set) |

**Warning event:** `FailedScheduling — pod has unbound immediate PersistentVolumeClaims`
- The pod was briefly unschedulable because Kubernetes first had to provision the PVC before it could place the pod on a node. The PVC became `Bound` within seconds and scheduling succeeded.

---

### 5. Exec into `nginx-sts-2` — write files
```powershell
kubectl exec nginx-sts-2 -it -- /bin/sh
```
```sh
# cd var/www
# echo hello > hello.txt       # write to the PVC-backed volume
# cat hello.txt
hello

# cd /usr/share/nginx/html
# cat > index.html              # overwrite nginx's default page
Hello
# cat index.html
Hello
# exit
```
- `/var/www/` is the PVC mount — data written here survives pod restarts.
- `/usr/share/nginx/html/` is inside the container's ephemeral filesystem — data written here is lost if the pod is deleted.

---

### 6. Exec into `nginx-sts-0` — test stable DNS
```powershell
kubectl exec nginx-sts-0 -it -- /bin/sh
```
```sh
# curl http://nginx-sts-2.nginx-headless
Hello                           # ← served the custom index.html from nginx-sts-2

# curl http://nginx-sts-2
curl: (6) Could not resolve host: nginx-sts-2   # ← short name alone doesn't resolve
```
- `nginx-sts-2.nginx-headless` works because the headless service registers DNS entries for each pod in the form `<pod-name>.<service-name>`.
- `nginx-sts-2` alone fails — you must include the service name in the DNS lookup.

Full DNS pattern: `<pod>.<service>.<namespace>.svc.cluster.local`
Short form inside same namespace: `<pod>.<service>`

---

### 7. Delete pod `nginx-sts-2` — demonstrate persistence
```powershell
kubectl delete pod nginx-sts-2
# pod "nginx-sts-2" deleted
```
```powershell
kubectl get po
```
```
NAME          READY   STATUS    RESTARTS   AGE
nginx-sts-0   1/1     Running   0          11m
nginx-sts-1   1/1     Running   0          11m
nginx-sts-2   1/1     Running   0           9s   ← recreated with the same name
```
The StatefulSet controller immediately recreated the pod with the **same name** (`nginx-sts-2`) and **reattached the same PVC** (`www-nginx-sts-2`).

```powershell
kubectl exec nginx-sts-2 -it -- /bin/sh
# ls var/www
hello.txt                       # ← file written before deletion is still there
```
This is the core StatefulSet guarantee: **storage outlives the pod**.

---

### 8. Delete the StatefulSet and Service
```powershell
kubectl delete -f statefulset.yaml
# service "nginx-headless" deleted
# statefulset.apps "nginx-sts" deleted
```
Deleting the StatefulSet **does not delete the PVCs** — this is intentional to protect data. PVCs must be removed manually.

---

### 9. Delete PVCs manually
```powershell
kubectl delete pvc www-nginx-sts-o     # typo — letter 'o' instead of zero
# Error from server (NotFound): persistentvolumeclaims "www-nginx-sts-o" not found

kubectl delete pvc www-nginx-sts-0
kubectl delete pvc www-nginx-sts-1
kubectl delete pvc www-nginx-sts-2
```
The first attempt used the letter `o` instead of the digit `0`. The corrected commands deleted all three PVCs successfully.

---

## StatefulSet vs Deployment vs DaemonSet

| Feature | StatefulSet | Deployment | DaemonSet |
|---|---|---|---|
| Pod names | Stable, ordered (`-0`, `-1`) | Random hash | Random suffix |
| Storage | Dedicated PVC per pod | Shared or none | Shared or none |
| Pod identity | Sticky (same name on restart) | Interchangeable | Interchangeable |
| Startup/shutdown order | Ordered (0→1→2) | Parallel | Parallel |
| Headless service | Required for DNS | Not needed | Not needed |
| PVC deleted with pods | No — manual cleanup required | Yes (if used) | Yes (if used) |
| Typical use case | Databases, brokers | Stateless apps | Node-level agents |
