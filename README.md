# Kubernetes Hands-On Project Report

**Author:** Hermann N'zi

---

## Overview

This report documents a comprehensive hands-on exploration of Kubernetes using Docker Desktop as the local cluster. The project covered the full spectrum of Kubernetes fundamentals — from running a single pod, through networking, storage, configuration management, and cluster observability tools. Every topic was practiced through real YAML manifests applied to a live cluster, verified with `kubectl`, and cleaned up after each exercise.

---

## 1. Rolling Updates

**Directory:** `rollingUpdates/`

The project began with rolling updates — one of Kubernetes' most important deployment strategies. A deployment running `guybarrette/hello-app:2.0` was configured with:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

- `maxSurge: 1` allows one extra pod above the desired replica count during the update.
- `maxUnavailable: 1` allows one pod to be unavailable at a time.

This ensures zero-downtime deployments — old pods are replaced gradually rather than all at once.

---

## 2. ClusterIP Service

**Directory:** `clusterIp/`

This exercise demonstrated internal pod-to-pod communication using the default ClusterIP service type.

**Resources deployed:**
- A **Deployment** running 3 replicas of `nginx:alpine` labelled `app=app-example, env=prod`
- A **ClusterIP Service** exposing port 8080 externally mapped to container port 80
- A **busybox Pod** (`mybox`) used as a client

**Key test:** Exec-ed into `mybox` and ran:
```sh
wget -qO- http://svc-example:8080
```
The nginx welcome page was returned, proving that Kubernetes DNS resolved the service name and routed the request to one of the nginx pods — all without knowing any pod IP addresses.

**Takeaway:** ClusterIP services are cluster-internal only. They are the backbone of microservice-to-microservice communication inside Kubernetes.

---

## 3. NodePort Service

**Directory:** `nodeport/`

This exercise exposed an application to traffic from **outside the cluster** using a NodePort service.

**Resources deployed:**
- A **Deployment** running 2 replicas of `nginx:alpine` labelled `app=nginx, env=prod`
- A **NodePort Service** mapping external port `32410` → cluster port `80` → container port `80`

**Three-port mapping explained:**
| Port | Value | Role |
|------|-------|------|
| nodePort | 32410 | Opened on the node — accessible from the host machine |
| port | 80 | Internal ClusterIP port |
| targetPort | 80 | Port nginx listens on inside the container |

Unlike ClusterIP, the service could have been reached directly from a browser at `http://localhost:32410` without exec-ing into any pod.

---

## 4. LoadBalancer Service

**Directory:** `loadBalancer/`

This exercise demonstrated the LoadBalancer service type, which provisions an external load balancer and assigns a stable external IP.

**Resource deployed:**
- A **LoadBalancer Service** on port 8080 targeting container port 80, selecting `app=nginx, env=prod`

**Observed output:**
```
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
svc-example   LoadBalancer   10.97.160.92   localhost     8080:31837/TCP
```

On Docker Desktop, `localhost` is automatically assigned as the external IP. On a real cloud provider (AWS, GCP, Azure), this would trigger provisioning of an actual cloud load balancer with a public IP.

Notable: no pods matched the selector in this run, demonstrating that a service can exist independently of its backend pods. Also noted that `kubectl delete -f loadbalancer` (without `.yaml`) fails — the full filename is required.

---

## 5. Persistent Volumes

**Directory:** `volume/`

This exercise proved that data written inside a pod **survives pod deletion** using a PersistentVolume (PV) and PersistentVolumeClaim (PVC).

**Resources deployed:**
- **PV** (`pv001`): 10Mi `hostPath` volume backed by `/data/` on the node, `storageClassName: ssd`, reclaim policy `Retain`
- **PVC** (`myclaim`): Requested 10Mi of `ssd` storage — Kubernetes automatically bound it to `pv001`
- **Pod** (`mybox`): Mounted the claim at `/demo/` inside the container

**The persistence test:**
1. Created `hello.txt` inside `/demo/` in the first pod instance
2. Force-deleted the pod (`--force --grace-period=0`)
3. Recreated the pod
4. `/demo/hello.txt` was still there — data survived because it lived on the node's `/data/`, not in the container layer

**Key detail:** The `Retain` reclaim policy means the PV and its data persist even after the PVC is deleted — manual cleanup is required.

---

## 6. ConfigMaps

**Directory:** `configMaps/`

This exercise showed how to store non-sensitive configuration data in a ConfigMap and inject it into a pod as an environment variable.

**Resources deployed:**
- **ConfigMap** (`cm-example`): Stored two keys — `city: Ann Arbor` and `state: Michigan`
- **Pod** (`mybox`): Injected only the `city` key as env var `CITY`

**Key test inside the pod:**
```sh
echo $city    # empty — env vars are case-sensitive
echo $CITY    # Ann Arbor — correct
```

**Important lesson:** `kubectl describe` does not support the `-o` flag. The correct command to view raw YAML is:
```sh
kubectl get configmap cm-example -o yaml
```
This mistake was made three times in the session, reinforcing the distinction between `describe` and `get`.

---

## 7. Secrets

**Directory:** `secret/`

This exercise demonstrated how to store sensitive credentials in a Kubernetes Secret and inject them into a pod — keeping them out of application code and pod specs.

**Resources deployed:**
- **Secret** (`secrets`): Type `Opaque` with two base64-encoded keys — `username` and `password`
- **Pod** (`mybox`): Injected both keys as env vars `USERNAME` and `PASSWORD`

**Values stored:**
```
VGhlVXNlck5hbWU=  →  TheUserName
bXlwYXNzd29yZA==  →  mypassword
```

**Key test inside the pod:**
```sh
echo $USERNAME   # TheUserName
echo $PASSWORD   # mypassword
```

Kubernetes automatically base64-decoded the values before injecting them. `kubectl describe secret` only showed byte counts — not values — which is intentional to protect secrets from terminal or log exposure.

**Important distinction:** Base64 is encoding, not encryption. Real security comes from Kubernetes RBAC controlling who can read secrets, and from enabling encryption at rest on the cluster.

---

## 8. Liveness Probe

**Directory:** `livenessProbe/`

This exercise demonstrated Kubernetes' self-healing capability — automatically restarting a container when a liveness probe detects it has become unhealthy.

**Pod behaviour:** The container ran a deliberate fault-injection script:
```sh
touch /tmp/healthy    # healthy on startup
sleep 15              # simulates normal operation
rm -rf /tmp/healthy   # simulates the app going unhealthy
sleep 3600            # container keeps running but is now "broken"
```

**Probe configuration:**
```yaml
livenessProbe:
  exec:
    command: [cat, /tmp/healthy]
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```

**What happened — observed in real time across three `kubectl describe` runs:**

| Time | Event |
|------|-------|
| 0s | Container starts, `/tmp/healthy` created |
| 5s | First probe passes — healthy |
| 20s | Probe fails (file deleted) — failure 1/2 |
| 25s | Probe fails again — failure 2/2, threshold reached |
| 25s | Kubelet sends `Killing` event |
| ~26s | Container terminated with **exit code 137** (SIGKILL) |
| ~26s | Container restarts automatically — `Restart Count: 1` |

Exit code 137 always means the process was killed by SIGKILL — the signal Kubernetes sends when forcefully terminating a container.

---

## 9. Kubernetes Dashboard

**Directory:** `Dashboard/`

The Kubernetes Dashboard was set up as a browser-based GUI for visualising and managing cluster resources. It required:
- A **ServiceAccount** for authentication
- A **ClusterRoleBinding** granting the service account cluster-admin privileges
- A **hello-app** deployment (2 replicas) with a ClusterIP service — used as a sample workload to observe inside the dashboard

The dashboard provides a visual alternative to `kubectl` for browsing pods, deployments, services, and events.

---

## 10. Lens IDE

**Directory:** `lens/`

Lens is a desktop Kubernetes IDE that connects directly to the cluster and provides a rich visual interface. A 3-replica deployment of `guybarrette/hello-app:1.0` with a RollingUpdate strategy was used as the sample workload to explore inside Lens.

Lens offers real-time pod logs, resource editing, shell access, and metrics — making it a powerful alternative to running repeated `kubectl` commands.

---

## 11. K9s

**Directory:** `K9s/`

K9s is a terminal-based Kubernetes UI that provides a fast, keyboard-driven interface for navigating cluster resources. The same hello-app deployment was used as the sample workload.

K9s is particularly useful for quickly switching between namespaces, watching pod restarts live, and tailing logs — all from the terminal without leaving the keyboard.

---

## 12. Pod Scaling and HPA

**Directory:** `scalePods/`

This exercise covered both manual scaling and **Horizontal Pod Autoscaling (HPA)** — automatically scaling pods based on CPU usage.

**Resources deployed:**
- **Metrics Server** (`components.yaml`): Required by HPA to collect CPU/memory metrics from pods
- **php-apache Deployment** with CPU `requests: 200m` and `limits: 500m` — a CPU-intensive sample app
- **ClusterIP Service** exposing the php-apache deployment

HPA watches the metrics server and automatically increases or decreases the replica count when CPU usage crosses a defined threshold — without manual intervention.

---

## 13. Blue/Green Deployments

**Directory:** `BandG/`

This exercise demonstrated a **Blue/Green deployment** strategy — running two versions of an application simultaneously and switching traffic between them by updating a service selector.

**Resources:**
- `hello-dep-v1.yaml` — Deployment running `guybarrette/hello-app:1.0` labelled `app: hello-v1` (Blue)
- `hello-dep-v2.yaml` — Deployment running `guybarrette/hello-app:2.0` labelled `app: hello-v2` (Green)
- `clusterip.yaml` — A ClusterIP service (`svc-front`) whose selector points to `app: hello-v1`

**How the switch works:** To cut traffic from v1 to v2, only the service selector needs to change from `app: hello-v1` to `app: hello-v2`. Both deployments remain running, allowing instant rollback by switching the selector back.

This is safer than a rolling update because both versions are live at the same time and the switch is instantaneous.

---

## Tools and Environment

| Tool | Purpose |
|------|---------|
| **Docker Desktop** | Single-node Kubernetes cluster on Windows |
| **kubectl** | Primary CLI for applying manifests and inspecting cluster state |
| **Kubernetes Dashboard** | Browser-based GUI for resource visualisation |
| **Lens** | Desktop IDE for Kubernetes cluster management |
| **K9s** | Terminal UI for fast keyboard-driven cluster navigation |

---

## Key Lessons Learned

1. **`kubectl describe` does not support `-o`** — use `kubectl get <resource> -o yaml` for formatted output.
2. **Env var names are case-sensitive** — `$CITY` works, `$city` does not.
3. **Base64 is encoding, not encryption** — Secrets require RBAC and encryption-at-rest for real security.
4. **ClusterIP, NodePort, and LoadBalancer** are a hierarchy — each builds on the previous and adds external reach.
5. **PVs outlive pods** — data written via a PVC persists through pod deletion and recreation.
6. **Exit code 137** always means SIGKILL — the container was force-terminated, not crashed.
7. **Liveness probes enable self-healing** — Kubernetes restarts unhealthy containers automatically without human intervention.
8. **Blue/Green deployments** offer instant rollback by keeping both versions live and redirecting the service selector.
9. **`kubectl delete -f`** requires the full filename with extension — not a partial path or directory.
