# Multi-Container Pods — Kubernetes Lab

This lab demonstrates how to run multiple containers inside a single Kubernetes Pod, and how containers within the same Pod share the same network — meaning they can reach each other via `localhost`.

---

## How Multi-Container Pods Work

All containers in a Pod share:
- **The same network namespace** — they communicate via `localhost`
- **The same IP address** — only one IP is assigned to the Pod
- **The same lifecycle** — they are scheduled, started, and stopped together

```
Pod: two-containers  (IP: 10.1.0.14)
┌─────────────────────────────────────────┐
│                                         │
│  Container: mynginx                     │
│  image: nginx       ← port 80           │
│  serves HTTP at localhost:80            │
│                         ↑              │
│                    localhost            │
│                         ↓              │
│  Container: mybox                       │
│  image: busybox     ← port 81           │
│  runs: sleep 3600                       │
│  can wget localhost to reach nginx      │
│                                         │
└─────────────────────────────────────────┘
```

---

## The Manifest — `two-containers.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Always
  containers:
  - name: mynginx
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
  - name: mybox
    image: busybox
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
    ports:
      - containerPort: 81
    command:
      - sleep
      - "3600"
```

**Field-by-field breakdown:**

| Field | Purpose |
|---|---|
| `restartPolicy: Always` | Kubernetes restarts any container in this Pod if it exits, regardless of exit code |
| `containers` (list of 2) | Both containers are peers — neither is an init container; they start in parallel |
| `mynginx` / `image: nginx` | Runs the nginx web server, listening on port 80 |
| `mybox` / `image: busybox` | Minimal Linux utility container |
| `command: sleep 3600` | Keeps the busybox container alive for 1 hour — without this it would exit immediately since busybox has no long-running default process |
| `containerPort: 81` | Declares busybox listens on port 81 (informational; no actual service binds there) |

---

## Step-by-Step: What Was Done

### Step 1 — Create the Pod

```powershell
kubectl create -f two-containers.yaml
```

**Result:** `pod/two-containers created`

---

### Step 2 — Verify Both Containers are Running

```powershell
kubectl get pods -o wide
```

**Output:**
```
NAME             READY   STATUS    RESTARTS   AGE   IP          NODE
two-containers   2/2     Running   0          30s   10.1.0.14   docker-desktop
```

**What `2/2` means:** 2 out of 2 containers are ready and running. The Pod has a single IP (`10.1.0.14`) shared by both containers.

---

### Step 3 — Inspect the Pod in Detail

```powershell
kubectl describe pod two-containers
```

Key details from the output:

**Container: `mynginx`**
- Image: `nginx`
- Port: `80/TCP`
- State: Running (started at 21:37:20)

**Container: `mybox`**
- Image: `busybox`
- Port: `81/TCP`
- Command: `sleep 3600`
- State: Running (started at 21:37:26)

**Events (startup sequence):**

| Time (relative) | Event |
|---|---|
| 0s | Pod scheduled to `docker-desktop` node |
| 6s | Pulled `nginx` image (6.19s) |
| 6s | Created and started `mynginx` container |
| 16s | Pulled `busybox` image (3.82s) |
| 22s | Created and started `mybox` container |

The containers started **sequentially** here because `busybox` had to be pulled after `nginx` — but in general, regular containers in a Pod start in parallel (unlike init containers, which are strictly sequential).

**QoS Class: Burstable** — both containers set `requests` lower than `limits`, so Kubernetes classifies the Pod as Burstable (not Guaranteed or BestEffort).

---

### Step 4 — Open a Shell in the `mybox` Container

```powershell
kubectl exec -it two-containers --container mybox -- /bin/sh
```

The `--container mybox` flag is required here because the Pod has more than one container. Without it, kubectl would default to the first container (`mynginx`), which does not have `sh` in an easily accessible shell form for this test.

---

### Step 5 — First `wget` Attempt (Flag Typo — Failed)

```sh
wget -q0- localhost
```

**Error:**
```
wget: invalid option -- '0'
```

**Why it failed:** The flag was typed as `-q0-` (with a zero) instead of `-qO-` (with a capital letter O). In BusyBox's `wget`:
- `-q` = quiet mode (no progress output)
- `-O -` = output to stdout (the `-` means stdout)

The digit `0` is not a valid wget flag.

---

### Step 6 — Correct `wget` (Capital O)

```sh
wget -qO- localhost
```

**Output:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.</p>
...
</html>
```

This confirms the core concept of the lab: **the `mybox` container reached the `mynginx` container via `localhost`**. Both containers share the same network namespace inside the Pod, so port 80 on `localhost` inside `mybox` is the same port 80 that nginx is listening on inside `mynginx`.

---

### Step 7 — Exit the Shell

```sh
exit
```

---

### Step 8 — Force Delete the Pod

```powershell
kubectl delete -f two-containers.yaml --force --grace-period=0
```

**Output:**
```
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated.
The resource may continue to run on the cluster indefinitely.
pod "two-containers" force deleted
```

**What these flags do:**

| Flag | Effect |
|---|---|
| `--force` | Skips the normal graceful termination signal (`SIGTERM`) |
| `--grace-period=0` | Sets the shutdown wait time to 0 seconds instead of the default 30s |

**The warning** is important: Kubernetes marks the Pod as deleted in its API immediately, but the containers on the node may still be running briefly. In production, force deletion should be avoided unless a Pod is stuck in `Terminating` state.

---

## Key Concepts Demonstrated

| Concept | Observation |
|---|---|
| **Shared network namespace** | Containers in the same Pod reach each other via `localhost` — no Service or IP needed |
| **`2/2 READY`** | Shows how many containers in the Pod are running, not how many Pods |
| **`sleep 3600`** | Required to keep busybox alive — containers with no long-running process exit immediately |
| **`--container` flag** | Required with `kubectl exec` when a Pod has more than one container |
| **`-qO-` vs `-q0-`** | Capital letter `O` (output to stdout) vs digit `0` — a common typo with BusyBox wget |
| **`restartPolicy: Always`** | Kubernetes restarts any container that exits; without `sleep`, busybox would restart in a loop |
| **`--force --grace-period=0`** | Immediately removes the Pod from the API without waiting for graceful shutdown |
| **QoS: Burstable** | Assigned when `requests` < `limits`; the Pod can burst above its guaranteed minimum |
