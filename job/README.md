# Kubernetes Job

## What is a Kubernetes Job?

A **Job** creates one or more Pods and ensures they run to **completion** (not continuously like a Deployment). Once all Pods finish successfully, the Job is done. It is used for one-off or batch tasks — things you run once and expect to finish.

## The Manifest (`job.yaml`)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["echo", "Hello from the Job"]
      restartPolicy: Never
```

| Field | Value | Meaning |
|-------|-------|---------|
| `kind` | `Job` | A run-to-completion workload, not a long-running service |
| `image` | `busybox` | Lightweight Linux utility image |
| `command` | `echo "Hello from the Job"` | The single command the container runs |
| `restartPolicy` | `Never` | Do not restart the container if it exits — the Job handles retries instead |

## Commands Walkthrough

### 1. Create the Job
```powershell
kubectl apply -f job.yaml
# job.batch/hello created
```
Kubernetes schedules a Pod (`hello-s47wt`) to run the command.

### 2. Check Job status
```powershell
kubectl get jobs
# NAME    COMPLETIONS   DURATION   AGE
# hello   1/1           7s         17s
```
`1/1` means 1 completion required, 1 completed. The Job finished in 7 seconds.

### 3. Inspect the Job details
```powershell
kubectl describe job hello
```
Shows the full Job spec, timing (started 15:25:20, completed 15:25:27), and the event log — including the Pod that was created (`hello-s47wt`) and the `Job completed` event.

### 4. Check the Pod
```powershell
kubectl get pods
# NAME          READY   STATUS      RESTARTS   AGE
# hello-s47wt   0/1     Completed   0          74s
```
The Pod has `Completed` status — it ran, finished, and stopped. It is no longer running (`0/1 READY`) but is kept so you can retrieve its logs.

### 5. View the output
```powershell
kubectl logs hello-s47wt
# Hello from the Job
```
The Pod printed `Hello from the Job` to stdout, which is captured as logs.

### 6. Delete the Job
```powershell
kubectl delete -f job.yaml
# job.batch "hello" deleted
```
Deleting the Job also deletes its completed Pod (`hello-s47wt`). After this, `kubectl get pods` returns nothing.

### 7. Trying to delete again
```powershell
kubectl delete -f job.yaml
# Error from server (NotFound): jobs.batch "hello" not found
```
Expected error — the Job was already deleted, so there is nothing to remove.

## Key Concepts

- **Job vs Deployment**: A Deployment keeps Pods running forever (restarts on exit). A Job runs Pods until they succeed, then stops.
- **Completed Pods are kept**: After the Job finishes, the Pod remains in `Completed` state so you can read its logs. It is cleaned up when you delete the Job.
- **`restartPolicy: Never`**: Tells Kubernetes not to restart the container on exit. Use `OnFailure` if you want automatic retries on failure.
