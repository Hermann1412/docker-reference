# Selectors — Kubernetes Lab

This lab demonstrates how **label selectors** connect a Kubernetes Service to a Pod. It shows what happens when labels match (traffic is routed) and what happens when they don't (the endpoint disappears).

---

## How Selectors Work

A Service does not know about Pods by name. It finds Pods exclusively by matching their **labels** against its **selector**. If every key-value pair in the selector matches a Pod's labels, the Pod becomes an endpoint of the Service.

```
Pod labels:                    Service selector:
  app: myapp        ✔ match      app: myapp
  type: front-end   ✔ match      type: front-end
                              → Pod IP added to endpoints ✔

Pod labels:                    Service selector:
  app: myapp2       ✗ no match   app: myapp
  type: front-end   ✔ match      type: front-end
                              → No endpoints ✗
```

---

## The Manifests

### `myapp.yaml` — Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp2       # <-- label value is myapp2
    type: front-end
spec:
  containers:
  - name: nginx-container
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

### `myservice.yaml` — Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: myapp        # <-- selector looks for myapp, not myapp2
    type: front-end
```

**The mismatch:** The Pod has `app: myapp2` but the Service selector requires `app: myapp`. Both must match exactly for the Service to route traffic to the Pod.

| Field | Pod label | Service selector | Match? |
|---|---|---|---|
| `app` | `myapp2` | `myapp` | No |
| `type` | `front-end` | `front-end` | Yes |

---

## Step-by-Step: What Was Done

### Step 1 — Navigate to the Folder (with Typo)

```powershell
cd selector
```

**Error:**
```
Set-Location: Cannot find path '...\docker\selector' because it does not exist.
```

The folder is named `selectors` (plural), not `selector`. Fixed immediately:

```powershell
cd selectors
```

---

### Step 2 — Deploy the Pod

```powershell
kubectl apply -f myapp.yaml
```

**Result:** `pod/myapp-pod created`

The Pod starts with labels `app: myapp2` and `type: front-end`, running an nginx container on port 80.

---

### Step 3 — Deploy the Service

```powershell
kubectl apply -f myservice.yaml
```

**Result:** `service/myservice created`

The Service is created with selector `app: myapp` and `type: front-end`, listening on port 80 and forwarding to `targetPort: 80`.

---

### Step 4 — Check Pod Details

```powershell
kubectl get po -o wide
```

**Output:**
```
NAME        READY   STATUS    RESTARTS   AGE     IP          NODE
myapp-pod   1/1     Running   0          2m53s   10.1.0.13   docker-desktop
```

`-o wide` shows extra columns including the Pod's internal cluster IP (`10.1.0.13`). The Pod is healthy and running.

---

### Step 5 — Check Service Endpoints

```powershell
kubectl get ep myservice
```

**Output:**
```
NAME        ENDPOINTS      AGE
myservice   10.1.0.13:80   100s
```

The Service found the Pod and added it as an endpoint at `10.1.0.13:80`.

**Wait — why does it match here if `app: myapp2` ≠ `app: myapp`?**

At this point the Pod was created from the *current state of `myapp.yaml`* which has `app: myapp2`. The endpoint showing `10.1.0.13:80` suggests the Pod was initially deployed with `app: myapp` (matching the Service), and `myapp.yaml` was edited to `myapp2` *between* the first apply and the second apply in Step 8. The endpoint confirms traffic was routing successfully at this stage.

---

### Step 6 — Forward Local Port to the Service

```powershell
kubectl port-forward service/myservice 8080:80
```

**Output:**
```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

`port-forward` creates a tunnel from `localhost:8080` on the local machine to port 80 of the Service inside the cluster. A browser or `curl` pointed at `http://localhost:8080` would reach the nginx Pod. The line `Handling connection for 8080` confirms at least one request came through.

---

### Step 7 — Re-apply the Pod Manifest (Label Changed)

```powershell
kubectl apply -f myapp.yaml
```

**Result:** `pod/myapp-pod configured`

`configured` (not `created`) means the resource already existed and was updated. At this point `myapp.yaml` had `app: myapp2` — the label was changed. Kubernetes updated the Pod's labels in-place.

---

### Step 8 — Check Endpoints Again (Now Empty)

```powershell
kubectl get ep myservice
```

**Output:**
```
NAME        ENDPOINTS   AGE
myservice   <none>      7m24s
```

The endpoint is gone. As soon as the Pod's label changed from `app: myapp` to `app: myapp2`, it no longer matched the Service selector (`app: myapp`). Kubernetes immediately removed it from the endpoint list. The Service still exists but has no Pods to route traffic to.

**This is the core lesson of the lab:** selectors must match labels exactly. A single character difference (`myapp` vs `myapp2`) breaks the connection between a Service and its Pods.

---

### Step 9 — Delete Service (Missing `.yaml` — Failed)

```powershell
kubectl delete -f myservice
```

**Error:**
```
error: the path "myservice" does not exist
```

`kubectl delete -f` expects a file path, not a resource name. The `.yaml` extension was missing.

---

### Step 10 — Delete the Service (Correct)

```powershell
kubectl delete -f myservice.yaml
```

**Result:** `service "myservice" deleted`

---

### Step 11 — Delete the Pod

```powershell
kubectl delete -f myapp.yaml
```

**Result:** `pod "myapp-pod" deleted`

---

## Key Concepts Demonstrated

| Concept | Observation |
|---|---|
| **Label selectors** | A Service routes traffic only to Pods whose labels match all selector key-value pairs exactly |
| **Endpoint controller** | Kubernetes automatically adds/removes Pod IPs from a Service's endpoint list as labels change |
| **`kubectl get ep`** | Shows the actual IPs and ports the Service is currently routing to — `<none>` means no matching Pods |
| **`kubectl apply` update** | Re-applying a manifest with changed fields updates the live resource in-place (`configured`) |
| **`kubectl port-forward`** | Tunnels a local port to a Service or Pod — useful for testing without exposing via LoadBalancer or NodePort |
| **`-o wide`** | Adds extra columns to `kubectl get` output, including Pod IP and node name |
| **`-f` requires a file** | `kubectl delete -f` takes a file path — omitting the extension causes an error |
