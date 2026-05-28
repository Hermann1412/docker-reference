# Kubernetes DaemonSet — Hands-on Session

## What is a DaemonSet?

A **DaemonSet** ensures that exactly **one pod runs on every node** in the cluster (or a filtered subset of nodes). When a new node joins, the DaemonSet controller automatically schedules a pod on it. When a node is removed, the pod is garbage-collected.

Typical real-world uses:
- Log collectors (Fluentd, Filebeat)
- Node monitoring agents (Prometheus Node Exporter, Datadog)
- Network plugins (CNI, kube-proxy)
- Storage daemons (Ceph, GlusterFS)

DaemonSets have **no `replicas` field** — the desired count is always equal to the number of (matching) nodes.

---

## Why only 1 pod here?

This cluster runs on **Docker Desktop**, which has a single node (`docker-desktop`). One node = one pod. In a production cluster with 10 nodes you would see 10 pods, each on a different node.

---

## Manifest — `daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-example
  labels:
    app: daemonset-example
spec:
  selector:
    matchLabels:
      app: daemonset-example
  template:
    metadata:
      labels:
        app: daemonset-example
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: busybox
        image: busybox
        args:
        - sleep
        - "10000"
```

| Field | Value | Meaning |
|---|---|---|
| `selector.matchLabels` | `app=daemonset-example` | DaemonSet owns pods with this label |
| `image` | `busybox` | Minimal Linux image (~1 MB) |
| `args` | `sleep 10000` | Keeps the container alive with no real workload |
| `tolerations` | `node-role.kubernetes.io/master: NoSchedule` | Allows the pod to be scheduled on master/control-plane nodes, which normally repel workloads |

**No `replicas` field** — this is intentional and correct for a DaemonSet.

---

## Tolerations explained

By default, control-plane nodes carry a **taint** (`NoSchedule`) that prevents regular pods from landing on them. A **toleration** is the pod's opt-in to ignore a specific taint. Without this toleration, the DaemonSet pod would be blocked from running on the master node.

```
Taint on node   →  node-role.kubernetes.io/master:NoSchedule
Toleration in pod  →  key: node-role.kubernetes.io/master, effect: NoSchedule
Result             →  pod is allowed to schedule on that node
```

---

## Commands Executed

### 0. Typo — wrong filename
```powershell
kubectl apply -f daemonet.yaml
# error: the path "daemonet.yaml" does not exist
```
The filename was misspelled (`daemonet` instead of `daemonset`). No resources were created — `kubectl apply` exits immediately when the file is not found.

---

### 1. Apply the manifest
```powershell
kubectl apply -f daemonset.yaml
# daemonset.apps/daemonset-example created
```
The DaemonSet controller inspects all nodes and schedules one pod per node. Single node → one pod created.

---

### 2. List pods (default, then wide)
```powershell
kubectl get pods
```
```
NAME                      READY   STATUS    RESTARTS   AGE
daemonset-example-xbds5   1/1     Running   0          13s
```

```powershell
kubectl get pods -o wide
```
```
NAME                      READY   STATUS    RESTARTS   AGE   IP          NODE
daemonset-example-xbds5   1/1     Running   0          84s   10.1.0.25   docker-desktop
```

- Pod name follows `<daemonset-name>-<random-suffix>` — no ReplicaSet hash, because DaemonSets don't use ReplicaSets.
- `1/1 Running` — container healthy, no restarts.
- `-o wide` confirms the pod landed on `docker-desktop`, the only node.

---

### 3. Delete the DaemonSet
```powershell
kubectl delete -f daemonset.yaml
# daemonset.apps "daemonset-example" deleted
```
Deleting the DaemonSet removes all pods it manages across every node.

---

## DaemonSet vs ReplicaSet vs Deployment

| Feature | DaemonSet | ReplicaSet | Deployment |
|---|---|---|---|
| Controls pod count | 1 per node (automatic) | Fixed replica count | Fixed replica count |
| `replicas` field | None | Required | Required |
| Rolling updates | Yes | No | Yes |
| Rollback | Yes | No | Yes |
| Typical use case | Node-level agents | Rarely used directly | Stateless applications |
| Pod naming | `<ds>-<suffix>` | `<rs>-<suffix>` | `<deploy>-<rs-hash>-<suffix>` |
