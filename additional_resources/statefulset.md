# StatefulSet

A StatefulSet is a Kubernetes controller designed for stateful applications. Unlike a Deployment, where pods are considered interchangeable and disposable, a StatefulSet gives each pod a **stable, unique identity** — a predictable name, a stable DNS hostname, and its own persistent storage — that survives pod restarts and rescheduling.

It is preferred for applications that require stable identities, such as databases (e.g., MySQL, PostgreSQL) or distributed systems (e.g., Kafka, Cassandra, Redis Cluster).

## StatefulSet vs Deployment

| Property | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix (`nginx-6b66bfb4f-hfk8v`) | Ordered index (`web-0`, `web-1`, `web-2`) |
| Pod identity | Interchangeable | Unique and stable |
| Storage | Shared or none | Each pod gets its own PVC |
| Startup order | All at once | Ordered: `web-0` → `web-1` → `web-2` |
| Shutdown order | All at once | Reverse order: `web-2` → `web-1` → `web-0` |
| DNS | Service IP (load-balanced) | Per-pod DNS via headless service |

Use a Deployment when your app is stateless and any pod can serve any request.
Use a StatefulSet when each pod needs a stable identity, its own storage, or must start/stop in a defined order.

## Core Concepts

### Stable Pod Names

StatefulSet pods are named `<statefulset-name>-<ordinal>`, always starting from `0`. A StatefulSet named `web` with 3 replicas produces pods named `web-0`, `web-1`, and `web-2`. If `web-1` crashes and is rescheduled, it will always come back with the same name and the same PVC — not a new random identity.

### Headless Service

A StatefulSet requires a **headless service** (`clusterIP: None`) to control the DNS domain of its pods. Unlike a normal service which provides a single virtual IP that load-balances across pods, a headless service creates individual DNS records for each pod:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For the example below, the DNS entries would be:
- `web-0.nginx.default.svc.cluster.local`
- `web-1.nginx.default.svc.cluster.local`
- `web-2.nginx.default.svc.cluster.local`

This is how distributed applications (e.g., a database cluster) allow members to address each other directly by a stable, predictable hostname.

### volumeClaimTemplates

Instead of mounting a single shared volume, a StatefulSet uses `volumeClaimTemplates` to automatically create a separate PersistentVolumeClaim for each pod. Each PVC is named `<template-name>-<pod-name>`:

- `www-web-0`
- `www-web-1`
- `www-web-2`

These PVCs are not deleted when the StatefulSet is scaled down or deleted, preserving the data for potential reuse.

### Ordered Deployment and Scaling

By default, pods are created one at a time in ascending order and each must be Running and Ready before the next one starts. Scaling down removes pods in reverse order.

This ordered behavior is critical for applications like databases where a primary must be running before replicas can join.

## Manifest

See: [nginx_statefulset.yaml](../examples/statefulset/nginx_statefulset.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  clusterIP: None  # Headless service — no virtual IP, DNS resolves directly to pod IPs
  selector:
    app: nginx
  ports:
  - port: 80
    name: web
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"  # Must match the headless service name above
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.29.0
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
```

## Hands-on

### Creating the StatefulSet

Apply the manifest (it includes both the headless service and the StatefulSet):

```bash
kubectl apply -f examples/statefulset/nginx_statefulset.yaml
```

```text
service/nginx created
statefulset.apps/web created
```

### Observing Ordered Startup

Watch the pods come up — notice they appear one at a time in order:

```bash
kubectl get pods -w
```

```text
NAME    READY   STATUS              RESTARTS   AGE
web-0   0/1     ContainerCreating   0          3s
web-0   1/1     Running             0          8s
web-1   0/1     ContainerCreating   0          9s
web-1   1/1     Running             0          14s
web-2   0/1     ContainerCreating   0          15s
web-2   1/1     Running             0          20s
```

`web-1` only starts after `web-0` is Running and Ready, and `web-2` only starts after `web-1`.

Check the final state — note the predictable names and the distribution across nodes:

```bash
kubectl get pods -o wide
```

```text
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE
web-0   1/1     Running   0          2m    10.244.0.5    minikube
web-1   1/1     Running   0          2m    10.244.1.6    minikube-m02
web-2   1/1     Running   0          2m    10.244.2.7    minikube-m03
```

Inspect the StatefulSet:

```bash
kubectl get statefulset web
```

```text
NAME   READY   AGE
web    3/3     2m
```

```bash
kubectl describe statefulset web
```

### Persistent Volume Claims

Each pod got its own PVC automatically:

```bash
kubectl get pvc
```

```text
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-a1b2c3d4-1111-2222-3333-aabbccddeeff   1Gi        RWO            standard       2m
www-web-1   Bound    pvc-b2c3d4e5-2222-3333-4444-bbccddeegg00   1Gi        RWO            standard       2m
www-web-2   Bound    pvc-c3d4e5f6-3333-4444-5555-ccddeeff1111   1Gi        RWO            standard       2m
```

### Stable DNS

You can verify the per-pod DNS records using a temporary pod. Run a busybox container to look up the headless service:

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx
```

```text
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.244.2.7 web-2.nginx.default.svc.cluster.local
Address 2: 10.244.1.6 web-1.nginx.default.svc.cluster.local
Address 3: 10.244.0.5 web-0.nginx.default.svc.cluster.local
```

A normal (non-headless) service returns a single ClusterIP. A headless service returns all pod IPs, each with its individual DNS name. You can also resolve a specific pod directly:

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-0.nginx
```

```text
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.0.5 web-0.nginx.default.svc.cluster.local
```

### Storage Persistence Demo

This is the key feature of StatefulSets — storage survives pod deletion. Let's prove it:

**Step 1**: Write a unique page to `web-0`'s volume.

```bash
kubectl exec web-0 -- sh -c 'echo "Hello from web-0 storage" > /usr/share/nginx/html/index.html'
```

**Step 2**: Confirm the page is served:

```bash
kubectl exec web-0 -- curl -s localhost
```

```text
Hello from web-0 storage
```

**Step 3**: Delete the pod (simulating a crash or rescheduling):

```bash
kubectl delete pod web-0
```

```text
pod "web-0" deleted
```

**Step 4**: Wait for the StatefulSet to recreate it (watch the pod come back with the same name):

```bash
kubectl get pods -w
```

```text
NAME    READY   STATUS        RESTARTS   AGE
web-0   1/1     Terminating   0          5m
web-1   1/1     Running       0          5m
web-2   1/1     Running       0          5m
web-0   0/1     Pending       0          2s
web-0   0/1     ContainerCreating 0      3s
web-0   1/1     Running       0          8s
```

**Step 5**: Verify the data survived:

```bash
kubectl exec web-0 -- curl -s localhost
```

```text
Hello from web-0 storage
```

The data is intact because the pod reconnected to the same PVC (`www-web-0`).

Contrast this with `web-1` and `web-2`, which still serve the default nginx page — each pod has its own independent storage:

```bash
kubectl exec web-1 -- curl -s localhost | grep title
```

```text
<title>Welcome to nginx!</title>
```

### Scaling

Scale up to 5 replicas — the new pods `web-3` and `web-4` appear in order:

```bash
kubectl scale statefulset web --replicas=5
```

```text
statefulset.apps/web scaled
```

```bash
kubectl get pods -w
```

```text
NAME    READY   STATUS              RESTARTS   AGE
web-0   1/1     Running             0          10m
web-1   1/1     Running             0          10m
web-2   1/1     Running             0          10m
web-3   0/1     ContainerCreating   0          4s
web-3   1/1     Running             0          9s
web-4   0/1     ContainerCreating   0          10s
web-4   1/1     Running             0          15s
```

Scale back down to 3 — pods are removed in reverse order (`web-4` first, then `web-3`):

```bash
kubectl scale statefulset web --replicas=3
```

```bash
kubectl get pods -w
```

```text
NAME    READY   STATUS        RESTARTS   AGE
web-4   1/1     Terminating   0          2m
web-4   0/1     Terminating   0          2m
web-3   1/1     Terminating   0          2m
web-3   0/1     Terminating   0          2m
web-0   1/1     Running       0          12m
web-1   1/1     Running       0          12m
web-2   1/1     Running       0          12m
```

The PVCs for `web-3` and `web-4` are **not** deleted automatically:

```bash
kubectl get pvc
```

```text
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-a1b2c3d4-...                           1Gi        RWO            standard       12m
www-web-1   Bound    pvc-b2c3d4e5-...                           1Gi        RWO            standard       12m
www-web-2   Bound    pvc-c3d4e5f6-...                           1Gi        RWO            standard       12m
www-web-3   Bound    pvc-d4e5f6g7-...                           1Gi        RWO            standard       12m
www-web-4   Bound    pvc-e5f6g7h8-...                           1Gi        RWO            standard       12m
```

This is intentional — Kubernetes preserves data in case you scale back up and want `web-3` to reattach to its previous data. If you no longer need them, delete manually:

```bash
kubectl delete pvc www-web-3 www-web-4
```

### Rolling Updates

When you update the pod template (e.g., a new image), the StatefulSet performs a rolling update in reverse order by default — starting with the highest ordinal pod to minimize disruption:

Update the image in the manifest to `nginx:1.28.0` and apply:

```bash
kubectl apply -f examples/statefulset/nginx_statefulset.yaml
```

Watch the update:

```bash
kubectl rollout status statefulset web
```

```text
Waiting for partitioned roll out to finish: 0 out of 3 new pods have been updated...
Waiting for 1 pods to be ready...
Waiting for partitioned roll out to finish: 1 out of 3 new pods have been updated...
Waiting for 1 pods to be ready...
Waiting for partitioned roll out to finish: 2 out of 3 new pods have been updated...
Waiting for 1 pods to be ready...
statefulset rolling update complete 3 pods at revision web-...
```

The pods are updated `web-2` → `web-1` → `web-0`. Each pod must be Running and Ready before the next is updated.

To rollback, restore the original image in the manifest and re-apply.

### Cleanup

```bash
kubectl delete statefulset web
kubectl delete service nginx
```

Note: Deleting the StatefulSet does **not** delete its PVCs. Delete them explicitly if you no longer need the data:

```bash
kubectl delete pvc -l app=nginx
```

## When Not to Use a StatefulSet

Not every persistent workload needs a StatefulSet. If your application:
- Uses a single database instance (not a cluster), a Deployment with a single PVC is simpler.
- Is truly stateless but reads from an external database, use a Deployment.
- Needs persistent storage but pods are interchangeable, a Deployment with a shared `ReadWriteMany` volume (e.g., NFS) may be more appropriate.

StatefulSets add operational complexity (PVC lifecycle, ordered operations). Only use them when the stable identity or ordered operations are genuinely required.

## Known limitation: resizing volumeClaimTemplates storage

A long-standing limitation of StatefulSets is that the `volumeClaimTemplates` field is effectively immutable after creation. Suppose you want to grow storage from `1Gi` to `2Gi` and update the manifest accordingly:

```yaml
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 2Gi  # changed from 1Gi
```

Applying this produces an error:

```bash
kubectl apply -f examples/statefulset/nginx_statefulset.yaml
```

```text
The StatefulSet "web" is invalid: spec.volumeClaimTemplates: Forbidden: updates to statefulset spec for fields other than 'replicas', 'ordinals', 'template', 'updateStrategy', 'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden.
```

This is tracked in [kubernetes/kubernetes#68737](https://github.com/kubernetes/kubernetes/issues/68737), open since 2018. The root cause is that StatefulSet validation was never extended to allow `volumeClaimTemplates` changes even after PVC resizing became a supported feature in Kubernetes v1.11.

**The practical impact:** if your database outgrows its initial storage allocation, you cannot simply edit the StatefulSet manifest and apply it.

**Workaround:** revert the manifest back to `1Gi` (so `kubectl apply` stops failing), then resize each PVC directly:

```bash
kubectl edit pvc www-web-0
# change spec.resources.requests.storage to 2Gi
```

Repeat for each pod's PVC (`www-web-1`, `www-web-2`, etc.). The StatefulSet itself does not need to be touched — the pods will automatically use the newly sized volumes once the underlying storage provider completes the resize. Note that this requires a StorageClass with `allowVolumeExpansion: true`.

The consequence is that the manifest will still say `1Gi` while the actual PVCs on the cluster are `2Gi`. However, you can reconcile the discrepancy by deleting and recreating the StatefulSet — PVCs survive a StatefulSet deletion, so the recreated StatefulSet will simply adopt the existing (already resized) ones:

```bash
# 1. Delete the StatefulSet — pods are removed, PVCs are left intact
kubectl delete statefulset web

# 2. Update the manifest to the new size (2Gi) and apply
kubectl apply -f examples/statefulset/nginx_statefulset.yaml
```

The StatefulSet comes back, the pods reconnect to their existing PVCs (now 2Gi), and the manifest matches the cluster state. Use `--cascade=orphan` if you want to keep pods running during the transition and avoid any downtime:

```bash
kubectl delete statefulset web --cascade=orphan
# pods keep running while you apply the updated manifest
kubectl apply -f examples/statefulset/nginx_statefulset.yaml
```

With `--cascade=orphan` the StatefulSet is deleted but its pods are left running as unmanaged orphans. Once you apply the updated manifest the new StatefulSet controller claims them back and reconciles normally.

## Further Reading

- [Kubernetes StatefulSets documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Running a Replicated Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)
- [StatefulSet Basics tutorial](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
- [Issue #68737 — StatefulSet volumeClaimTemplates resize](https://github.com/kubernetes/kubernetes/issues/68737)
