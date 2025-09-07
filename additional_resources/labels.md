# How labels work in Kubernetes

Assuming you have a Pod and replicaSet defined like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    app: nginx-rs
spec:
  replicas: 3 # Desired number of Pods
  selector: # Matches labels on the Pods it should manage
    matchLabels:
      app: nginx-rs
  template: # This is the Pod template for the ReplicaSet
    metadata:
      labels:
        app: nginx-rs
    spec:
      containers:
      - name: nginx
        image: nginx:1.29.0
```

We can see that the ReplicaSet will create three Pods with the label `app: nginx-rs`. The nginx Pod resource will run one additional nginx Pod.

When you run `kubectl get pods`, you will see something like this:

```text
NAME          READY   STATUS             RESTARTS   AGE    IP            NODE              NOMINATED NODE   READINESS GATES
nginx         1/1     Running            0          4m3s   10.244.2.9    desktop-worker    <none>           <none>
nginx-77jhk   1/1     Running            0          4m3s   10.244.1.10   desktop-worker2   <none>           <none>
nginx-g6dmv   1/1     Running            0          4m3s   10.244.2.11   desktop-worker    <none>           <none>
nginx-jkcww   1/1     Running            0          4m3s   10.244.2.10   desktop-worker    <none>           <none>
```

But if we add a label `app: nginx-rs` to the Pod metadata the ReplicaSet will then manage this Pod as well. As a result of that one pod will be destroyed as the replica count is set to 3 and at this moment we have 4 Pods.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx-rs
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
```

After applying this change, if you run `kubectl get pods` again, you will see that the ReplicaSet has adjusted the number of Pods to match the desired state:

```text
NAME          READY   STATUS             RESTARTS   AGE    IP            NODE              NOMINATED NODE   READINESS GATES
nginx-77jhk   1/1     Running            0          4m9s   10.244.1.10   desktop-worker2   <none>           <none>
nginx-g6dmv   1/1     Running            0          4m9s   10.244.2.11   desktop-worker    <none>           <none>
nginx-jkcww   1/1     Running            0          4m9s   10.244.2.10   desktop-worker    <none>           <none>
```

NOTE: It is up to Kubernetes ReplicaSet controller to decide which Pod to delete. It can be any of the Pods that are managed by the ReplicaSet, including the one that was created manually.

So the result could also look like this if the controller decided differently:

```text
NAME          READY   STATUS             RESTARTS   AGE    IP            NODE              NOMINATED NODE   READINESS GATES
nginx         1/1     Running            0          4m3s   10.244.2.9    desktop-worker    <none>           <none>
nginx-77jhk   1/1     Running            0          4m3s   10.244.1.10   desktop-worker2   <none>           <none>
nginx-g6dmv   1/1     Running            0          4m3s   10.244.2.11   desktop-worker    <none>           <none>
```
