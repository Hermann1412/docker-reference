# Kubernetes CronJob

## What is a Kubernetes CronJob?

A **CronJob** creates a Job on a repeating schedule — just like a Unix cron task. Each time it fires, it spins up a Pod, runs the command, and marks it complete. It is used for recurring tasks: backups, report generation, cleanup scripts, etc.

## The Manifest (`cronjob.yaml`)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: busybox
            image: busybox
            command: ["echo", "Hello from the CronJob"]
          restartPolicy: Never
```

| Field | Value | Meaning |
|-------|-------|---------|
| `kind` | `CronJob` | A scheduled, repeating workload |
| `schedule` | `* * * * *` | Run every minute (all five fields are `*`) |
| `image` | `busybox` | Lightweight Linux utility image |
| `command` | `echo "Hello from the CronJob"` | The command each Pod runs |
| `restartPolicy` | `Never` | Do not restart the container on exit |

### Cron schedule format
```
* * * * *
│ │ │ │ └─ day of week (0–7)
│ │ │ └─── month (1–12)
│ │ └───── day of month (1–31)
│ └─────── hour (0–23)
└───────── minute (0–59)
```
`* * * * *` means "every minute, every hour, every day."

## Commands Walkthrough

### 1. Create the CronJob
```powershell
kubectl apply -f cronjob.yaml
# cronjob.batch/hello-cron created
```

### 2. Check CronJob status
```powershell
kubectl get cronjob
# NAME         SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# hello-cron   * * * * *   False     0        10s             15s
```
`ACTIVE 0` means no Job is running right now. `LAST SCHEDULE` shows when the last Job was triggered.

### 3. Typo — `cornjob` does not exist
```powershell
kubectl describe cornjob.yaml   # wrong — typo + wrong syntax
kubectl describe cornjob        # wrong — still a typo
# error: the server doesn't have a resource type "cornjob"
```
The correct resource type is `cronjob`, not `cornjob`.

### 4. Describe the CronJob (correct)
```powershell
kubectl describe cronjob
kubectl describe cronjob hello-cron   # same result, explicit name
```
Key fields from the output:

| Field | Value | Meaning |
|-------|-------|---------|
| `Schedule` | `* * * * *` | Fires every minute |
| `Concurrency Policy` | `Allow` | A new Job can start even if the previous one is still running |
| `Successful Job History Limit` | `3` | Only keep the last 3 completed Jobs (and their Pods) |
| `Failed Job History Limit` | `1` | Only keep the last 1 failed Job |
| `Active Jobs` | varies | Jobs currently running |

The Events section showed a new Job being created and completing every minute:
```
SuccessfulCreate  → Created job hello-cron-29668839
SawCompletedJob   → Saw completed job: hello-cron-29668839, status: Complete
SuccessfulCreate  → Created job hello-cron-29668840
...
```

### 5. Why `kubectl logs` kept failing

```powershell
kubectl get pods
# hello-cron-29668840-824x2   0/1   Completed   0   2m50s  ← copied this name

kubectl logs hello-cron-29668840-824x2
# Error from server (NotFound): pods "hello-cron-29668840-824x2" not found
```

This happened **every single time** for the same reason:

1. `kubectl get pods` listed 3 Completed pods.
2. You copied a pod name.
3. In the time it took to type `kubectl logs ...`, the next minute fired.
4. A new Job completed, pushing the oldest pod past the **`successfulJobsHistoryLimit: 3`** threshold.
5. Kubernetes automatically deleted the oldest pod to stay within the limit.
6. The pod name you copied no longer existed.

The fix is to use a label selector so Kubernetes finds *whatever pod is currently there*:
```powershell
kubectl logs -l job-name=hello-cron --tail=20
```

### 6. Delete the CronJob
```powershell
kubectl delete -f cronjob.yaml
# cronjob.batch "hello-cron" deleted
```
Deleting the CronJob also deletes all its Jobs and Pods immediately.

```powershell
kubectl get pods
# No resources found in default namespace.
```

## Job vs CronJob vs Deployment

| | Deployment | Job | CronJob |
|-|------------|-----|---------|
| **Purpose** | Long-running service | One-off task | Repeated scheduled task |
| **Restarts on exit** | Yes | No | No (each run is a fresh Job) |
| **Ends on its own** | No | Yes (on success) | Yes (each Job ends) |
| **Schedule** | No | No | Yes (cron expression) |

## Key Takeaway

`successfulJobsHistoryLimit: 3` (the default) means Kubernetes keeps only the **3 most recent completed Pods**. Since the CronJob fired every minute, pods were being deleted roughly every minute too — you need to read logs quickly, or use a label-based query instead of a hardcoded pod name.
