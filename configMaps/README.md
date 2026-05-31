# Kubernetes ConfigMap Demo

This demo shows how a **ConfigMap** stores non-sensitive configuration data and injects it into a pod as an environment variable.

---

## Files

| File | Kind | Purpose |
|------|------|---------|
| `cm.yaml` | ConfigMap | Stores two key-value pairs: `city` and `state` |
| `pod.yaml` | Pod | Reads the `city` key from the ConfigMap and exposes it as the env var `CITY` |

---

## What Each File Does

### `cm.yaml` — ConfigMap
```yaml
kind: ConfigMap
metadata:
  name: cm-example
data:
  state: Michigan
  city: Ann Arbor
```
- Stores plain text configuration as key-value pairs — no encoding, no encryption.
- Two keys: `state` and `city`.
- Any pod in the same namespace can reference this ConfigMap by name (`cm-example`).
- Use ConfigMaps for non-sensitive config (URLs, feature flags, region names). For passwords and tokens, use a **Secret** instead.

### `pod.yaml` — Pod
```yaml
env:
  - name: CITY
    valueFrom:
      configMapKeyRef:
        name: cm-example
        key: city
```
- Injects only the `city` key from `cm-example` into the container as an environment variable named `CITY`.
- The env var name (`CITY`) does not have to match the ConfigMap key (`city`) — you choose the name.
- `state` was defined in the ConfigMap but not mounted in this pod — you only inject what you need.

---

## What You Did Step by Step

### 1. Created the ConfigMap
```powershell
kubectl apply -f cm.yaml
kubectl get cm
```
Output showed `cm-example` with **2 data keys**:
```
NAME         DATA   AGE
cm-example   2      8s
```

### 2. Inspected the ConfigMap contents
```powershell
kubectl describe configmap cm-example
```
Output confirmed both keys and their values:
```
Data
====
city:   Ann Arbor
state:  Michigan
```

### 3. Three failed attempts to get YAML output from describe
```powershell
kubectl describe configmap cm-example -o YAML   # error: unknown shorthand flag 'o'
kubectl describe configmap cm-example -O YAML   # error: unknown shorthand flag 'O'
kubectl describe configmap cm-example -o YAML   # same error again
```
`kubectl describe` does not support the `-o` flag — that flag belongs to `kubectl get`, not `describe`. The correct command to view raw YAML would be:
```powershell
kubectl get configmap cm-example -o yaml
```

### 4. Created the pod
```powershell
kubectl apply -f pod.yaml
kubectl exec mybox -it -- /bin/sh
```

### 5. Tested the environment variable inside the pod
```sh
echo $city    # printed nothing — env var names are case-sensitive
echo $CITY    # printed "Ann Arbor" — correct
exit
```
The lowercase `$city` returned empty because Linux environment variables are **case-sensitive** — the pod defined it as `CITY` (uppercase), so only `$CITY` works.

### 6. Cleaned up
```powershell
kubectl delete -f cm.yaml    # deleted configmap
kubectl delete -f pod.yaml   # deleted pod
```

---

## Key Concepts Demonstrated

| Concept | What happened |
|---------|---------------|
| **ConfigMap as config store** | Plain key-value pairs stored in Kubernetes, separate from the pod spec |
| **Selective injection** | Only `city` was injected as an env var — `state` was ignored by this pod |
| **Key-to-envvar mapping** | ConfigMap key `city` was mapped to env var `CITY` — names can differ |
| **Case sensitivity** | `$city` returned nothing; `$CITY` returned "Ann Arbor" — env vars are case-sensitive in Linux |
| **`describe` vs `get -o yaml`** | `describe` is human-readable but does not support `-o`; use `kubectl get ... -o yaml` for raw YAML output |
| **ConfigMap vs Secret** | ConfigMaps are for non-sensitive data only — values are stored in plain text |
