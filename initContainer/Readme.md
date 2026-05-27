# Init Containers — Kubernetes Lab

This lab demonstrates how **init containers** work in Kubernetes. An init container runs to completion before the main application container starts, and can be used to prepare the environment — in this case, downloading a webpage that Nginx will serve.

---

## How Init Containers Work

```
Pod starts
    │
    ▼
┌─────────────────────────────┐
│  Init Container: install    │  ← runs first, must complete successfully
│  image: busybox             │
│  wget http://info.cern.ch   │  ← downloads index.html into shared volume
│  → writes to /work-dir/     │
└─────────────────────────────┘
    │  (only starts after init completes)
    ▼
┌─────────────────────────────┐
│  Main Container: nginx      │  ← starts after init is done
│  image: nginx               │
│  serves /usr/share/nginx/html│ ← reads index.html from same shared volume
└─────────────────────────────┘
```

The two containers communicate through a shared **emptyDir** volume — init writes to it, nginx reads from it.

---

## The Manifest — `myapp.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
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
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: install
    image: busybox
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://info.cern.ch
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  volumes:
  - name: workdir
    emptyDir: {}
```

**Field-by-field breakdown:**

| Field | Purpose |
|---|---|
| `kind: Pod` | Creates a single Pod (not a Deployment — no auto-restart or scaling) |
| `containers[].name: nginx` | The main app container; serves HTTP on port 80 |
| `containers[].volumeMounts.mountPath: /usr/share/nginx/html` | Nginx default web root — whatever is here gets served |
| `initContainers[].name: install` | The init container; runs before nginx starts |
| `initContainers[].image: busybox` | Minimal Linux image that includes `wget` |
| `command: wget -O /work-dir/index.html http://info.cern.ch` | Downloads the CERN homepage and saves it as `index.html` |
| `initContainers[].volumeMounts.mountPath: /work-dir` | The same volume, mounted at a different path in the init container |
| `volumes[].emptyDir: {}` | A temporary shared directory created fresh when the Pod starts; deleted when the Pod is removed |

---

## Step-by-Step: What Was Done

### Step 1 — Deploy the Pod

```powershell
kubectl apply -f myapp.yaml
```

**Result:** `pod/init-demo created`

`kubectl apply` is the declarative way to create or update resources from a YAML file.

---

### Step 2 — Check Pod Status (Init Container Still Running)

```powershell
kubectl get pods
```

**Output:**
```
NAME        READY   STATUS     RESTARTS   AGE
init-demo   0/1     Init:0/1   0          21s
```

**What the status means:**

| Column | Value | Meaning |
|---|---|---|
| `READY` | `0/1` | 0 out of 1 main containers are ready — nginx hasn't started yet |
| `STATUS` | `Init:0/1` | 0 out of 1 init containers have completed — `install` is still running (downloading the file) |
| `RESTARTS` | `0` | No failures |

The Pod is not yet serving traffic. Kubernetes is waiting for the init container to finish downloading the page.

---

### Step 3 — Verify the Running Container via Docker

```powershell
docker ps
```

**Output:**
```
CONTAINER ID   IMAGE   COMMAND                  CREATED          STATUS
f0427982bfb0   nginx   "/docker-entrypoint.…"   13 seconds ago   Up 11 seconds
```

This confirms that by the time `docker ps` was run, the init container had already completed and the nginx container was up. The container name includes the full Kubernetes metadata: `k8s_nginx_init-demo_default_...`.

---

### Step 4 — Open a Shell Inside the Running Pod

```powershell
kubectl exec -it init-demo -- /bin/bash
```

**Output:**
```
Defaulted container "nginx" out of: nginx, install (init)
```

`kubectl exec` opens an interactive shell (`-it`) inside the Pod. Since the Pod has two containers (nginx and the completed install init container), kubectl defaulted to the **nginx** container — the only one currently running. The `install` init container had already exited.

---

### Step 5 — Curl the Nginx Server from Inside the Container

```bash
curl localhost
```

**Output:**
```html
<html><head></head><body><header>
<title>http://info.cern.ch</title>
</header>
<h1>http://info.cern.ch - home of the first website</h1>
...
</body></html>
```

This confirms the full init container flow worked:

1. The `install` init container downloaded `http://info.cern.ch` and saved it to the shared volume at `/work-dir/index.html`
2. The nginx container mounted the same volume at `/usr/share/nginx/html`
3. Nginx is now serving that downloaded file as its default page
4. `curl localhost` (port 80) returns the CERN website — the first website ever published on the World Wide Web

---

### Step 6 — Exit the Shell

```bash
exit
```

Returns to the local PowerShell session.

---

### Step 7 — Delete the Pod

```powershell
kubectl delete -f myapp.yaml
```

**Result:** `pod "init-demo" deleted`

Using `-f myapp.yaml` deletes all resources defined in that file. The Pod, its containers, and the `emptyDir` volume (and the downloaded file inside it) are all permanently removed.

---

## Key Concepts Demonstrated

| Concept | Observation |
|---|---|
| **Init container sequencing** | Kubernetes guarantees the init container runs to completion before the main container starts |
| **Shared emptyDir volume** | The only way init and main containers communicate — both mount the same volume at different paths |
| **`Init:0/1` status** | Shows how many init containers have completed out of how many total |
| **busybox image** | Minimal utility image used for setup tasks; includes common Unix tools like `wget` |
| **nginx default web root** | `/usr/share/nginx/html` — whatever files are placed here are served by nginx |
| **`kubectl exec -it`** | Opens an interactive terminal inside a running container |
| **`kubectl apply` vs `kubectl create`** | `apply` creates or updates; `create` only creates (fails if resource exists) |
| **`kubectl delete -f`** | Deletes all resources defined in the manifest, not just one by name |
