# L18-02 — Kubernetes Namespaces & kubectl Basics

This lab explores Kubernetes namespaces: how to list them, switch the active namespace in a context, create a new one, and delete it. It also covers common kubectl command errors.

---

## What is a Namespace?

A namespace is a virtual cluster inside a Kubernetes cluster. It isolates resources (Pods, Deployments, Services, etc.) so different teams or environments can coexist in the same cluster without interfering with each other.

### Built-in Namespaces

| Namespace | Purpose |
|---|---|
| `default` | Where resources land if no namespace is specified |
| `kube-system` | Core Kubernetes components (API server, scheduler, etc.) |
| `kube-public` | Publicly readable resources; used for cluster info |
| `kube-node-lease` | Heartbeat records for each node to detect failures |

---

## Step-by-Step: What Was Done

### Step 1 — List All Namespaces (Full and Shorthand)

```powershell
kubectl get namespaces
kubectl get ns
```

**Output (both commands):**
```
NAME              STATUS   AGE
default           Active   25h
kube-node-lease   Active   25h
kube-public       Active   25h
kube-system       Active   25h
```

`ns` is the official shorthand for `namespaces` in kubectl. Both commands produce identical output. The cluster had been running for 25 hours at this point.

---

### Step 2 — Check Pods in the Default Namespace

```powershell
kubectl get pods
```

**Output:**
```
No resources found in default namespace.
```

With no namespace flag specified, kubectl targets the **active namespace** from the current context, which is `default`. No user workloads had been deployed yet.

---

### Step 3 — Typo: `kube` Instead of `kubectl`

```powershell
kube get pods -n kube-system
```

**Error:**
```
kube: The term 'kube' is not recognized as a name of a cmdlet, function, script file, or executable program.
```

**Why it failed:** The command-line tool is `kubectl` (Kubernetes control), not `kube`. PowerShell could not find any executable named `kube`.

---

### Step 4 — List Pods in `kube-system` Namespace

```powershell
kubectl get pods -n kube-system
```

**Output:**
```
NAME                                     READY   STATUS    RESTARTS      AGE
coredns-5dd5756b68-68s7r                 1/1     Running   1 (22h ago)   25h
coredns-5dd5756b68-7thcr                 1/1     Running   1 (22h ago)   25h
etcd-docker-desktop                      1/1     Running   1 (22h ago)   25h
kube-apiserver-docker-desktop            1/1     Running   1 (22h ago)   25h
kube-controller-manager-docker-desktop   1/1     Running   2 (22h ago)   25h
kube-proxy-6hnbl                         1/1     Running   1 (22h ago)   25h
kube-scheduler-docker-desktop            1/1     Running   4 (15h ago)   25h
storage-provisioner                      1/1     Running   4 (15h ago)   25h
vpnkit-controller                        1/1     Running   1 (22h ago)   25h
```

The `-n` flag targets a specific namespace without changing the active context. These are the core system components that keep Kubernetes itself running:

| Pod | Role |
|---|---|
| `coredns` (×2) | DNS resolution inside the cluster |
| `etcd` | The cluster's key-value store — holds all cluster state |
| `kube-apiserver` | The front door to the cluster; all kubectl commands go through it |
| `kube-controller-manager` | Reconciliation loops (ensures desired state matches actual state) |
| `kube-proxy` | Manages networking rules on each node for Service routing |
| `kube-scheduler` | Assigns new Pods to nodes based on resource availability |
| `storage-provisioner` | Handles dynamic volume provisioning (Docker Desktop specific) |
| `vpnkit-controller` | Manages host networking on Docker Desktop (Windows/Mac specific) |

The RESTARTS column shows `kube-scheduler` and `storage-provisioner` restarted 4 times — normal for a local Docker Desktop cluster that was likely suspended or restarted.

---

### Step 5 — Switch Active Namespace to `kube-system`

```powershell
kubectl config set-context --current --namespace=kube-system
```

**Result:** `Context "docker-desktop" modified.`

This permanently changes the **default namespace** of the current context (`docker-desktop`) to `kube-system`. From this point on, any `kubectl` command without a `-n` flag targets `kube-system` automatically.

---

### Step 6 — Confirm Namespace Switch Worked

```powershell
kubectl get pods
```

**Output:** The same 9 `kube-system` pods listed above — now returned without needing `-n kube-system`, confirming the context switch worked.

---

### Step 7 — Switch Active Namespace Back to `default`

```powershell
kubectl config set-context --current --namespace=default
```

**Result:** `Context "docker-desktop" modified.`

Restores the context to its original state so subsequent commands target the `default` namespace again.

---

### Step 8 — Confirm Switch Back

```powershell
kubectl get pods
```

**Output:** `No resources found in default namespace.` — confirms the active namespace is `default` again.

---

### Step 9 — Create a New Namespace

```powershell
kubectl create ns hello
```

**Result:** `namespace/hello created`

`ns` works as a shorthand here too. The new namespace `hello` is now available for isolating resources.

---

### Step 10 — Verify the New Namespace Appears

```powershell
kubectl get ns
```

**Output:**
```
NAME              STATUS   AGE
default           Active   25h
hello             Active   10s
kube-node-lease   Active   25h
kube-public       Active   25h
kube-system       Active   25h
```

`hello` appears with `AGE: 10s`, confirming it was just created. Status `Active` means it is ready to accept resources.

---

### Step 11 — Delete the Namespace

```powershell
kubectl delete ns hello
```

**Result:** `namespace "hello" deleted`

Deleting a namespace also deletes **all resources inside it** (Pods, Deployments, Services, etc.). This is a destructive operation — use with care in production.

---

### Step 12 — Confirm Namespace is Gone

```powershell
kubectl get ns
```

**Output:** The original 4 namespaces only — `hello` is no longer listed.

---

## Key Concepts Demonstrated

| Concept | Observation |
|---|---|
| **Namespace isolation** | Resources in one namespace are invisible to `kubectl get` targeting another |
| **`-n` flag** | Targets a specific namespace for a single command without changing context |
| **`set-context --namespace`** | Permanently changes the default namespace for all future commands in that context |
| **Namespace shorthands** | `ns` is shorthand for `namespaces` in all kubectl commands |
| **`kube-system` pods** | Core cluster components; always running, managed by Kubernetes itself |
| **Cascade deletion** | Deleting a namespace removes all resources inside it |
| **`kube` vs `kubectl`** | The CLI tool is always `kubectl` — `kube` alone is not a valid command |

---

## Reference: Useful Context Commands

```powershell
# Show the active context
kubectl config current-context

# List all available contexts
kubectl config get-contexts

# Switch to a different context
kubectl config use-context <contextName>

# Change default namespace in current context
kubectl config set-context --current --namespace=<namespace>

# Rename a context
kubectl config rename-context <old-name> <new-name>

# Delete a context
kubectl config delete-context <contextName>
```
