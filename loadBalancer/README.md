# Kubernetes LoadBalancer Service Demo

This demo shows how a **LoadBalancer** service exposes pods externally using a cloud-provided (or local) load balancer with a stable external IP.

---

## Files

| File | Kind | Purpose |
|------|------|---------|
| `loadbalancer.yaml` | Service (LoadBalancer) | Exposes nginx externally on port 8080 via `localhost` |

---

## What the File Does

### `loadbalancer.yaml` — LoadBalancer Service
```yaml
kind: Service
spec:
  type: LoadBalancer
  selector:
    app: nginx
    env: prod
  ports:
  - protocol: TCP
    port: 8080        # external port clients use
    targetPort: 80    # port nginx listens on inside the container
```
- `type: LoadBalancer` tells Kubernetes to provision an external load balancer.
- On **Docker Desktop**, it uses the built-in integration to assign `localhost` as the `EXTERNAL-IP` automatically — no cloud provider needed.
- On a real cloud (AWS, GCP, Azure), this would provision an actual cloud load balancer and assign a real public IP.
- Traffic hitting `localhost:8080` is forwarded to port 80 on any pod matching `app=nginx, env=prod`.
- No pods matched the selector in this demo, but the service itself was created and assigned its external IP successfully.

---

## What You Did Step by Step

### 1. Applied the LoadBalancer service
```powershell
kubectl apply -f loadbalancer.yaml   # created service/svc-example
```

### 2. Checked pods — none were running
```powershell
kubectl get pods -o wide
# No resources found in default namespace.
```
No deployment was applied in this demo, so there were no backend pods. The service was created but had nothing to route traffic to.

### 3. Inspected the service
```powershell
kubectl get svc -o wide
```
Output:
```
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE    SELECTOR
svc-example   LoadBalancer   10.97.160.92   localhost     8080:31837/TCP   98s    app=nginx,env=prod
```
Key things to notice:
- **CLUSTER-IP** `10.97.160.92` — the internal cluster IP (acts like ClusterIP internally)
- **EXTERNAL-IP** `localhost` — Docker Desktop automatically resolves this; on a cloud it would be a real public IP
- **PORT(S)** `8080:31837/TCP` — external port 8080 mapped to NodePort 31837 (Kubernetes allocates a NodePort automatically for LoadBalancer services too)
- **SELECTOR** `app=nginx,env=prod` — would match pods if a deployment had been applied

### 4. Attempted to delete with wrong path (minor mistake)
```powershell
kubectl delete -f loadbalancer        # error: path does not exist
kubectl delete -f loadbalancer.yaml   # correct — service deleted
```
`kubectl delete -f` requires the full filename, not just a directory name (unlike `kubectl apply -f <dir>/` which scans a directory).

---

## Key Concepts Demonstrated

| Concept | What happened |
|---------|---------------|
| **LoadBalancer type** | Kubernetes provisions an external load balancer and assigns an EXTERNAL-IP |
| **Docker Desktop behaviour** | Automatically sets EXTERNAL-IP to `localhost` — no cloud needed locally |
| **Auto NodePort** | Kubernetes also opened NodePort `31837` behind the scenes (`8080:31837`) |
| **No pods needed to create a service** | The service was created successfully even with no matching pods |
| **Selector without backends** | Service exists but traffic would return "no endpoints" until pods are deployed |

---

## LoadBalancer vs NodePort vs ClusterIP

| Feature | ClusterIP | NodePort | LoadBalancer |
|---------|-----------|----------|--------------|
| Internal cluster access | Yes | Yes | Yes |
| Host machine access | No | Yes (node port) | Yes (external IP) |
| External / internet access | No | Yes (if firewall allows) | Yes (cloud IP or localhost) |
| External IP assigned | No | No | Yes |
| Typical use case | Internal microservices | Dev/testing | Production external traffic |
