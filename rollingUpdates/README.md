# Kubernetes Rolling Updates

## What is a Rolling Update?

A **Rolling Update** replaces Pods one batch at a time — new Pods come up before old ones go down. This means your application stays available throughout the update with **zero downtime**. It is the default update strategy for Kubernetes Deployments.

## The Manifest (`hello-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-dep
  namespace: default
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: hello-dep
  template:
    metadata:
      labels:
        app: hello-dep
    spec:
      containers:
      - image: guybarrette/hello-app:2.0
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        imagePullPolicy: Always
        name: hello-dep
        ports:
        - containerPort: 8080
```

### Rolling Update strategy fields

| Field | Value | Meaning |
|-------|-------|---------|
| `maxSurge` | `1` | At most 1 extra Pod above the desired count can exist during the update |
| `maxUnavailable` | `1` | At most 1 Pod can be unavailable (not ready) during the update |

With 3 replicas, `maxSurge: 1` and `maxUnavailable: 1`, Kubernetes can have up to **4 Pods** temporarily (3 + 1 surge) and at least **2 Pods** running at all times during the rollout.

---

## Commands Walkthrough

### 1. Create the initial Deployment (v1.0)
```powershell
kubectl create -f hello-deployment.yaml
# deployment.apps/hello-dep created
```
> Note: The yaml file had `image: guybarrette/hello-app:1.0` at this point.

### 2. Check rollout status
```powershell
kubectl rollout status deployment/hello-dep
# deployment "hello-dep" successfully rolled out
```

### 3. Inspect the Pods (wide view)
```powershell
kubectl get pods -o wide
# NAME                         READY   STATUS    RESTARTS   AGE   IP
# hello-dep-57c986475d-54bbz   1/1     Running   0          71s   10.1.0.100
# hello-dep-57c986475d-fjzdg   1/1     Running   0          71s   10.1.0.99
# hello-dep-57c986475d-s7lst   1/1     Running   0          71s   10.1.0.98
```
All 3 Pods are from ReplicaSet `57c986475d` — the hash suffix identifies which ReplicaSet (and therefore which image version) they belong to.

### 4. Describe the Deployment
```powershell
kubectl describe deploy hello-dep
```
Key output:
- `StrategyType: RollingUpdate`
- `RollingUpdateStrategy: 1 max unavailable, 1 max surge`
- `Image: guybarrette/hello-app:1.0`
- `Revision: 1`

### 5. Check the ReplicaSet
```powershell
kubectl get rs
# NAME                   DESIRED   CURRENT   READY   AGE
# hello-dep-57c986475d   3         3         3       3m10s
```
One ReplicaSet managing all 3 Pods for version 1.0.

---

## Triggering the Rolling Update (v1.0 → v2.0)

The `hello-deployment.yaml` file was edited to change the image from `1.0` to `2.0`, then applied:

```powershell
kubectl apply -f hello-deployment.yaml
# deployment.apps/hello-dep configured
```

### Watching the rollout in k9s

The two screenshots below were taken during the rollout using **k9s**, a terminal UI for Kubernetes.

**Mid-rollout — old Pods terminating, new Pods starting:**

![Rolling Update in progress](Screenshot%202026-05-30%20172339.png)

3 new Pods (`hello-dep-65cb77d76f-*`) are coming up (Running/RunningΔ) while the old Pods were being replaced.

**Rollout complete — all new Pods running:**

![Rolling Update complete](Screenshot%202026-05-30%20172540.png)

All 3 Pods are now from the new ReplicaSet (`hello-dep-57c986475d-*`), all `1/1 Running`. The update is done.

---

### 6. Confirm rollout completed
```powershell
kubectl rollout status deployment/hello-dep
# deployment "hello-dep" successfully rolled out
```

### 7. Two ReplicaSets now exist
```powershell
kubectl get rs
# NAME                   DESIRED   CURRENT   READY   AGE
# hello-dep-57c986475d   0         0         0       8m     ← v1.0 (scaled down, kept for rollback)
# hello-dep-65cb77d76f   3         3         3       105s   ← v2.0 (active)
```
Kubernetes **keeps the old ReplicaSet** at 0 replicas so you can roll back instantly without re-pulling the image.

### 8. View rollout history
```powershell
kubectl rollout history deployment/hello-dep
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```
Two revisions exist. `CHANGE-CAUSE` is `<none>` because no `--record` flag or annotation was used.

> **Tip:** To populate CHANGE-CAUSE, add this annotation to the deployment metadata:
> ```yaml
> annotations:
>   kubernetes.io/change-cause: "updated to v2.0"
> ```

### 9. Rollback attempt
```powershell
kubectl rollout undo deployment/hello-dep --to-revision 1
# error: unable to find specified revision 1 in history
```
This failed because the Deployment was first created with `kubectl create` (not `kubectl apply --save-config`). When `kubectl apply` was later used for the update, it patched the annotation and reset the history, making revision 1 unreachable.

> **Fix:** Always use `kubectl apply -f` from the start (or `kubectl create --save-config`) to maintain a consistent history.

### 10. Rolling back without a target revision
```powershell
kubectl rollout undo deployment/hello-dep
```
This rolls back to the previous revision regardless of history numbering.

### 11. Delete the Deployment
```powershell
kubectl delete -f hello-deployment.yaml
# deployment.apps "hello-dep" deleted from default namespace
```

---

## Key Concepts

### Rolling Update vs Recreate

| Strategy | Behaviour | Downtime |
|----------|-----------|----------|
| `RollingUpdate` | Gradually replaces Pods — old and new run simultaneously | None |
| `Recreate` | Kills all old Pods first, then creates new ones | Yes |

### ReplicaSet history enables rollback

Every time the Pod template changes (e.g. new image), Kubernetes creates a **new ReplicaSet**. The old one is scaled to 0 but kept. This is what makes `kubectl rollout undo` instant — it just scales the old ReplicaSet back up.

### Typos encountered
```powershell
kunectl rollout ...   # typo — should be kubectl
get rs                # missing kubectl prefix
```
Both are common mistakes — always prefix with `kubectl`.
