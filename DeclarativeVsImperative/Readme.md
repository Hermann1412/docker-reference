# L18-04 — Declarative vs Imperative Deployments in Kubernetes

This lab demonstrates the two ways to manage Kubernetes resources: **imperative** (commands typed directly) and **declarative** (using a YAML manifest file).

---

## The Two Approaches

| Approach | How | Best for |
|---|---|---|
| **Imperative** | `kubectl create`, `kubectl delete` with flags | Quick one-off tasks, testing |
| **Declarative** | `kubectl create -f file.yaml` | Repeatable, version-controlled deployments |

---

## The YAML Manifest — `deploy-example.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynginx2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      env: prod
  template:
    metadata:
      labels:
        app: nginx
        env: prod
    spec:
      containers:
      - name: nginx
        image: nginx
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

**What each field means:**

| Field | Purpose |
|---|---|
| `apiVersion: apps/v1` | The Kubernetes API group and version for Deployments |
| `kind: Deployment` | The type of resource being created |
| `metadata.name: mynginx2` | The name that identifies this deployment in the cluster |
| `spec.replicas: 1` | Run exactly 1 Pod at all times |
| `selector.matchLabels` | The Deployment uses these labels to find and manage its Pods |
| `template.metadata.labels` | Labels stamped onto every Pod created by this Deployment — must match `matchLabels` |
| `image: nginx` | Pull the official Nginx image from Docker Hub |
| `resources.requests` | Minimum CPU/memory the scheduler must guarantee (`100m` CPU, `128Mi` RAM) |
| `resources.limits` | Maximum CPU/memory the container is allowed to use (`250m` CPU, `256Mi` RAM) |
| `containerPort: 80` | Documents that the container listens on port 80 (informational only) |

---

## Step-by-Step: What Was Done

### Step 1 — Navigate to the Lab Folder

```powershell
cd DeclarativeVsImperative
```

---

### Step 2 — Imperative Deployment

```powershell
kubectl create deployment mynginx --image=nginx
```

**Result:** `deployment.apps/mynginx created`

This is the **imperative** approach — the full desired state is expressed entirely in the command itself. Kubernetes receives an instruction and acts on it immediately. No file is needed, but nothing is saved or reproducible.

---

### Step 3 — Verify the Deployment is Running

```powershell
kubectl get deploy
```

**Output:**
```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
mynginx   1/1     1            1           48s
```

`1/1` under READY means 1 Pod was requested and 1 is running. The deployment was healthy 48 seconds after creation.

---

### Step 4 — Declarative Deployment (Typo — Failed)

```powershell
kubectl create -f deploy-exemple.yaml
```

**Error:**
```
error: the path "deploy-exemple.yaml" does not exist
```

**Why it failed:** The filename was misspelled — `exemple` instead of `example`. The file `deploy-example.yaml` exists in the folder but was not found because the name didn't match exactly.

---

### Step 5 — Declarative Deployment (Correct Filename)

```powershell
kubectl create -f deploy-example.yaml
```

**Result:** `deployment.apps/mynginx2 created`

This is the **declarative** approach — the desired state is fully described in the YAML file. Kubernetes reads the manifest and reconciles the cluster to match it. The file can be committed to version control and reapplied at any time.

---

### Step 6 — Verify Both Deployments are Running

```powershell
kubectl get deploy
```

**Output:**
```
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
mynginx    1/1     1            1           2m27s
mynginx2   1/1     1            1           21s
```

Both deployments are running simultaneously — one created imperatively, one declaratively.

---

### Step 7 — Delete `mynginx` (Typo — Failed)

```powershell
kubectl delete deployment myngix
```

**Error:**
```
Error from server (NotFound): deployments.apps "myngix" not found
```

**Why it failed:** The name was misspelled — `myngix` instead of `mynginx` (missing the `n`). Kubernetes looked for a deployment with that exact name and found nothing.

---

### Step 8 — Delete `mynginx` (Correct Name)

```powershell
kubectl delete deployment mynginx
```

**Result:** `deployment.apps "mynginx" deleted`

The imperative deployment was removed. Kubernetes immediately terminates the associated Pod.

---

### Step 9 — Delete `mynginx2`

```powershell
kubectl delete deploy mynginx2
```

**Result:** `deployment.apps "mynginx2" deleted`

The declarative deployment was removed. Note that `deploy` is a valid shorthand for `deployment` in kubectl — both forms work.

---

## Key Concepts Demonstrated

| Concept | Observation |
|---|---|
| **Imperative vs Declarative** | Both achieve the same result; declarative is preferred for production as it is reproducible and version-controlled |
| **YAML manifest** | Fully describes the desired state: replicas, image, labels, resource limits |
| **Resource requests vs limits** | `requests` = scheduler guarantee; `limits` = hard cap to prevent runaway containers |
| **Label selectors** | The Deployment tracks its Pods via `matchLabels` — labels on Pods must match exactly |
| **`kubectl get deploy`** | Quick way to check deployment health; `READY 1/1` means all replicas are up |
| **Typos are fatal** | Both a filename typo (`exemple`) and a resource name typo (`myngix`) caused failures |
| **`deploy` shorthand** | `kubectl delete deploy` is equivalent to `kubectl delete deployment` |

---

## Cleanup Commands

```powershell
kubectl delete deployment mynginx
kubectl delete deploy mynginx2
```
