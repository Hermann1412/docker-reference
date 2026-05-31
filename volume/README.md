# Kubernetes Persistent Volume Demo

This demo proves that data written inside a pod **survives pod deletion**. It uses a PersistentVolume (PV) and PersistentVolumeClaim (PVC) to give a pod durable storage that outlives any individual pod instance.

---

## Files

| File | Kind | Purpose |
|------|------|---------|
| `pv.yaml` | PersistentVolume | Provisions 10Mi of storage backed by the node's `/data/` directory |
| `pvc.yaml` | PersistentVolumeClaim | Claims that storage for use by a pod |
| `pod.yaml` | Pod | Mounts the claim at `/demo/` inside the container |

---

## What Each File Does

### `pv.yaml` — PersistentVolume
```yaml
kind: PersistentVolume
metadata:
  name: pv001
spec:
  storageClassName: ssd
  capacity:
    storage: 10Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/"
```
- Manually provisions a **10Mi** volume on the node at host path `/data/`.
- `storageClassName: ssd` is a custom label used to match this PV to a PVC requesting the same class.
- `accessModes: ReadWriteOnce` — only one node can mount it as read/write at a time.
- `persistentVolumeReclaimPolicy: Retain` — when the PVC is deleted, the PV and its data are **kept** (not wiped). You must manually clean it up.
- `hostPath` is a simple storage type suited for single-node clusters like Docker Desktop. On a real cluster you would use a cloud disk (EBS, GCE PD, etc.).

### `pvc.yaml` — PersistentVolumeClaim
```yaml
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
  storageClassName: ssd
```
- A **request** for storage — the pod never references the PV directly, only the claim.
- Kubernetes matched this claim to `pv001` because the `storageClassName`, `accessModes`, and `storage` size all aligned.
- Once bound, the PV status changed from `Available` → `Bound`.

### `pod.yaml` — Pod
```yaml
kind: Pod
metadata:
  name: mybox
spec:
  containers:
  - name: mybox
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
      - mountPath: "/demo/"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```
- The pod references the claim (`myclaim`) — not the PV directly.
- The claim's storage is mounted inside the container at `/demo/`.
- Anything written to `/demo/` is actually written to `/data/` on the node — outside the container's ephemeral filesystem.

---

## What You Did Step by Step

### 1. Typo on first attempt
```powershell
kubectl apply-f pv.yaml   # error: unknown command "apply-f"
kubectl apply -f pv.yaml  # correct — space between apply and -f
```

### 2. Created the PersistentVolume
```powershell
kubectl apply -f pv.yaml
kubectl get pv
```
PV `pv001` was created with status **Available** — waiting for a claim to bind to it.

### 3. Created the PersistentVolumeClaim
```powershell
kubectl apply -f pvc.yaml
kubectl get pv
```
After applying the PVC, `pv001` status changed to **Bound** — Kubernetes matched the claim to the PV automatically.
```
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS
pv001   10Mi       RWO            Retain           Bound    default/myclaim   ssd
```

### 4. Created the pod and wrote a file
```powershell
kubectl apply -f pod.yaml
kubectl exec mybox -it -- /bin/sh
```
Inside the shell:
```sh
ls              # saw "demo" directory — the volume is mounted
cd demo
cat > hello.txt # typed "Hello world" then Ctrl+D to save
ls              # confirmed hello.txt exists
exit
```

### 5. Force-deleted the pod
```powershell
kubectl delete -f pod.yaml --force --grace-period=0
```
- `--force --grace-period=0` skips the normal 30-second graceful termination and deletes immediately.
- The pod is gone — but the data is NOT gone, because it lives on the PV, not inside the container.

### 6. Recreated the pod and verified data survived
```powershell
kubectl apply -f pod.yaml
kubectl exec mybox -it -- /bin/sh
```
Inside the shell:
```sh
cd demo
ls              # hello.txt is still there
cat hello.txt   # printed "Hello world"
exit
```
This is the core proof: the file written in the first pod instance persisted across deletion and re-creation.

### 7. Cleaned up everything
```powershell
kubectl delete -f pod.yaml --force --grace-period=0
kubectl delete -f pvc.yaml   # releases the claim (PV stays due to Retain policy)
kubectl delete -f pv.yaml    # manually deletes the PV
```
Note: a second typo (`kubectl delete-f`) triggered another "unknown command" error — always include the space.

---

## Key Concepts Demonstrated

| Concept | What happened |
|---------|---------------|
| **PV / PVC separation** | Pod never references the PV directly — only the claim. This decouples storage provisioning from pod config |
| **Binding** | Kubernetes automatically matched the PVC to the PV based on storageClass, accessMode, and size |
| **Data persistence** | `hello.txt` survived pod deletion because it was stored on the node, not in the container layer |
| **hostPath** | Simple single-node storage — maps a node directory (`/data/`) into the container (`/demo/`) |
| **Retain policy** | Deleting the PVC does not wipe the PV — data is kept and must be cleaned up manually |
| **Force delete** | `--force --grace-period=0` bypasses graceful shutdown — useful in dev but avoid in production |
