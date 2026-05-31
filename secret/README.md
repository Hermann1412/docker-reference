# Kubernetes Secret Demo

This demo shows how a **Secret** stores sensitive data (credentials) and injects them into a pod as environment variables — keeping them out of your pod spec and application code.

---

## Files

| File | Kind | Purpose |
|------|------|---------|
| `secrets.yaml` | Secret | Stores base64-encoded `username` and `password` |
| `pod.yaml` | Pod | Injects both secret keys as env vars `USERNAME` and `PASSWORD` |

---

## What Each File Does

### `secrets.yaml` — Secret
```yaml
kind: Secret
type: Opaque
data:
  username: VGhlVXNlck5hbWU=
  password: bXlwYXNzd29yZA==
```
- `type: Opaque` is the generic secret type for arbitrary key-value data.
- Values **must be base64-encoded** in the YAML file — that is what Kubernetes expects in the `data` field.
- The encoding is not encryption — it is just encoding. Anyone with access to the secret can decode it instantly:
  ```
  VGhlVXNlck5hbWU=  →  TheUserName
  bXlwYXNzd29yZA==  →  mypassword
  ```
- Real security comes from Kubernetes RBAC restricting who can `get`/`list` secrets, and from enabling encryption at rest on the cluster.

### `pod.yaml` — Pod
```yaml
env:
  - name: USERNAME
    valueFrom:
      secretKeyRef:
        name: secrets
        key: username
  - name: PASSWORD
    valueFrom:
      secretKeyRef:
        name: secrets
        key: password
```
- Uses `secretKeyRef` (vs `configMapKeyRef` for ConfigMaps) to pull each key from the secret.
- Kubernetes automatically base64-decodes the value before injecting it — the pod sees the plain text `TheUserName`, not `VGhlVXNlck5hbWU=`.
- Both keys are injected here, unlike the ConfigMap demo where only one of two keys was used.

---

## What You Did Step by Step

### 1. Created the Secret
```powershell
kubectl apply -f secrets.yaml
kubectl get secret
```
Output showed `secrets` of type **Opaque** with **2 data keys**:
```
NAME      TYPE     DATA   AGE
secrets   Opaque   2      13s
```

### 2. Described the secret — values are hidden
```powershell
kubectl describe secret secrets
```
```
Data
====
password:  10 bytes
username:  11 bytes
```
`kubectl describe` intentionally hides the values — it only shows byte counts. This protects secrets from being exposed in logs or terminal recordings.

### 3. Two wrong attempts to get YAML output
```powershell
kubectl describe secret secrets -o YAML
# error: unknown shorthand flag 'o' in -o
# (describe does not support -o, same mistake as in the ConfigMap demo)

kubectl describe secret secrets o YAML
# Ran describe successfully but then tried to describe secrets named "o" and "YAML" — not found
```

### 4. Correct command to view raw YAML with encoded values
```powershell
kubectl get secret secrets -o YAML
```
This worked and showed the base64-encoded values in the YAML:
```yaml
data:
  password: bXlwYXNzd29yZA==
  username: VGhlVXNlck5hbWU=
```

### 5. Created the pod and verified secret injection
```powershell
kubectl apply -f pod.yaml
kubectl exec mybox -it -- /bin/sh
```
Inside the shell:
```sh
echo $USERNAME   # TheUserName
echo $PASSWORD   # mypassword
exit
```
Kubernetes decoded the base64 values automatically before injecting them — the pod saw plain text.

### 6. Cleaned up
```powershell
kubectl delete -f secrets.yaml   # deleted secret
kubectl delete -f pod.yaml       # deleted pod
```

---

## Key Concepts Demonstrated

| Concept | What happened |
|---------|---------------|
| **Secret vs ConfigMap** | Secrets are for sensitive data; ConfigMaps are for non-sensitive config |
| **Base64 encoding** | Values in `secrets.yaml` are base64-encoded — Kubernetes decodes them before injection |
| **Opaque type** | Generic secret type for arbitrary key-value pairs |
| **`describe` hides values** | Shows byte counts only — protects secrets from terminal/log exposure |
| **`get -o yaml` reveals encoded values** | Base64-encoded, not plain text — but trivially decodable |
| **`secretKeyRef`** | The pod-side reference to pull a specific key from a named secret |
| **`describe` does not support `-o`** | Use `kubectl get ... -o yaml` for formatted output, not `kubectl describe` |

---

## Secret vs ConfigMap Side by Side

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| For sensitive data | No | Yes |
| Values stored as | Plain text | Base64-encoded |
| `kubectl describe` shows values | Yes | No (byte count only) |
| Pod reference | `configMapKeyRef` | `secretKeyRef` |
| Encrypted at rest (default) | No | No (requires cluster config) |
| Use case | Config, URLs, flags | Passwords, tokens, keys |
