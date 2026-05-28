# Kubernetes ReplicaSet — Hands-on Session

## What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of identical pod replicas are running at any given time. If a pod dies, the ReplicaSet controller automatically creates a new one to maintain the desired count.

---

## Manifest — `rs-example.yaml`

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-example
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      type: front-end
  template:
    metadata:
      labels:
        app: nginx
        type: front-end
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
```

| Field | Value | Meaning |
|---|---|---|
| `replicas` | 3 | Keep 3 pods running at all times |
| `selector.matchLabels` | `app=nginx, type=front-end` | ReplicaSet owns pods with these labels |
| `image` | `nginx:alpine` | Lightweight nginx image |
| `cpu request / limit` | 100m / 250m | Guaranteed / max CPU per pod |
| `memory request / limit` | 128Mi / 256Mi | Guaranteed / max memory per pod |
| `containerPort` | 80 | Port the nginx process listens on inside the container |

---

## Commands Executed

### 1. Apply the manifest
```powershell
kubectl apply -f rs-example.yaml
# replicaset.apps/rs-example created
```
Kubernetes reads the YAML and creates the ReplicaSet object. The controller immediately spawns 3 pods.

---

### 2. Inspect pods (wide output)
```powershell
kubectl get pods -o wide
```
```
NAME               READY   STATUS              RESTARTS   AGE   IP       NODE
rs-example-b2ddr   0/1     ContainerCreating   0          27s   <none>   docker-desktop
rs-example-lhxgx   0/1     ContainerCreating   0          27s   <none>   docker-desktop
rs-example-p5spk   0/1     ContainerCreating   0          27s   <none>   docker-desktop
```
- Three pods with auto-generated suffix names (`b2ddr`, `lhxgx`, `p5spk`).
- Status `ContainerCreating` — image was being pulled from Docker Hub.
- `-o wide` adds the node name and IP columns.

---

### 3. Check the ReplicaSet
```powershell
kubectl get rs
```
```
NAME         DESIRED   CURRENT   READY   AGE
rs-example   3         3         3       69s
```
- **DESIRED** = what the spec asks for.
- **CURRENT** = pods that exist.
- **READY** = pods passing readiness checks.
All three matched — the ReplicaSet was healthy.

---

### 4. Describe the ReplicaSet
```powershell
kubectl describe rs rs-example
```
Key sections from the output:

| Section | Detail |
|---|---|
| Selector | `app=nginx, type=front-end` |
| Replicas | 3 current / 3 desired |
| Pods Status | 3 Running, 0 Waiting |
| Events | Controller emitted `SuccessfulCreate` for each of the 3 pods |

`describe` is the go-to command for debugging — it shows the full spec, current state, and the event log.

---

### 5. Delete the ReplicaSet
```powershell
kubectl delete -f rs-example.yaml
# replicaset.apps "rs-example" deleted
```
Passing the same manifest to `delete` tears down the ReplicaSet **and all pods it owns** in one step.

---

## Key Takeaways

- A ReplicaSet **self-heals**: delete a pod manually and the controller creates a replacement immediately.
- The **selector** is the glue — the ReplicaSet adopts any existing pod that matches its labels, even pods created before the ReplicaSet itself.
- `kubectl apply -f` / `kubectl delete -f` are the standard declarative workflow for creating and removing resources.
- Resource `requests` reserve capacity on the node; `limits` cap what the container can consume.
