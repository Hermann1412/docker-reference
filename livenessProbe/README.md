# Kubernetes Liveness Probe Demo

This demo shows how a **liveness probe** automatically detects when a container is unhealthy and restarts it — without any manual intervention.

---

## Files

| File | Kind | Purpose |
|------|------|---------|
| `pod.yaml` | Pod | Runs a container that deliberately becomes unhealthy after 15 seconds, triggering an automatic restart |

---

## What the File Does

### `pod.yaml` — Pod with Liveness Probe
```yaml
args:
- /bin/sh
- -c
- touch /tmp/healthy; sleep 15; rm -rf /tmp/healthy; sleep 3600
```
The container runs a script with four steps:
1. `touch /tmp/healthy` — creates the health file immediately on startup
2. `sleep 15` — waits 15 seconds (simulating a healthy app)
3. `rm -rf /tmp/healthy` — deletes the health file (simulates the app going unhealthy)
4. `sleep 3600` — hangs for an hour (the container keeps running but is now "broken")

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```
The probe settings:
- `exec` — runs `cat /tmp/healthy` inside the container to check health. Exit code 0 = healthy, non-zero = unhealthy.
- `initialDelaySeconds: 5` — waits 5 seconds after the container starts before running the first probe (gives the app time to initialise).
- `periodSeconds: 5` — re-runs the probe every 5 seconds.
- `failureThreshold: 2` — after **2 consecutive failures**, Kubernetes kills and restarts the container.

---

## Timeline of Events

This is the exact sequence that played out during the demo:

| Time | Event |
|------|-------|
| 0s | Container starts. `/tmp/healthy` is created immediately. |
| 5s | First probe runs — `cat /tmp/healthy` succeeds. Pod is healthy. |
| 15s | Container deletes `/tmp/healthy`. App is now "broken". |
| 20s | Probe runs — `cat /tmp/healthy` fails (file gone). Failure 1/2. |
| 25s | Probe runs again — still fails. Failure 2/2. Threshold reached. |
| 25s | Kubelet sends `Killing` event — container will be restarted. |
| ~26s | Container is terminated with exit code **137** (SIGKILL). |
| ~26s | Container restarts. `/tmp/healthy` is created again. Pod is healthy. |
| 80s+ | Cycle repeats — probe fails again after the new 15s window. |

---

## What You Did Step by Step

### 1. Created the pod
```powershell
kubectl apply -f pod.yaml
```

### 2. First describe — pod healthy, first probe failure just appeared
```powershell
kubectl describe pod liveness-example
```
Events showed the pod had just started and the first unhealthy event appeared:
```
Warning  Unhealthy  2s  kubelet  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```
`Restart Count: 0` — no restarts yet, only 1 failure so far (threshold is 2).

### 3. Second describe — threshold hit, container being killed
```powershell
kubectl describe pod liveness-example
```
Events now showed the probe had failed twice and the kubelet acted:
```
Warning  Unhealthy  25s (x2 over 30s)  kubelet  Liveness probe failed: ...
Normal   Killing    25s                 kubelet  Container liveness failed liveness probe, will be restarted
```
`Restart Count: 0` still — the restart hadn't completed yet.

### 4. Third describe — container restarted, new cycle beginning
```powershell
kubectl describe pod liveness-example
```
The container had now restarted. Key changes:
```
State:      Running
  Started:  Mon, 01 Jun 2026 02:14:02 +0700   ← new start time

Last State: Terminated
  Reason:   Error
  Exit Code: 137                               ← killed by SIGKILL
  Started:  Mon, 01 Jun 2026 02:13:07 +0700
  Finished: Mon, 01 Jun 2026 02:13:59 +0700

Restart Count: 1                               ← confirmed one restart
```
Exit code **137** means the process was killed by signal 9 (SIGKILL) — Kubernetes forcefully terminated the container when the liveness probe threshold was exceeded.

### 5. Force deleted the pod
```powershell
kubectl delete -f pod.yaml --force --grace-period=0
```

---

## Key Concepts Demonstrated

| Concept | What happened |
|---------|---------------|
| **Liveness probe** | Kubernetes ran `cat /tmp/healthy` every 5s to check if the container was alive |
| **exec probe** | Runs a command inside the container — exit 0 = healthy, non-zero = unhealthy |
| **failureThreshold** | After 2 consecutive failures, Kubernetes restarted the container automatically |
| **initialDelaySeconds** | 5-second grace period on startup before probing begins |
| **Exit code 137** | Container killed by SIGKILL (signal 9) — the normal exit code for a force-killed container |
| **Restart Count** | Kubernetes tracks how many times a container has been restarted in a pod's lifetime |
| **Self-healing** | The whole point — no human intervention needed, Kubernetes detected and fixed the fault automatically |

---

## Liveness Probe Types

This demo used an `exec` probe, but Kubernetes supports three types:

| Type | How it works | Use case |
|------|-------------|---------|
| `exec` | Runs a command inside the container | File checks, custom scripts |
| `httpGet` | Sends an HTTP GET and checks the response code | Web servers and APIs |
| `tcpSocket` | Opens a TCP connection to a port | Databases, raw TCP services |
