# Persistent Volume Binding

The binding of Persistent Volumes (PV) and Persistent Volume Claims (PVC) is a crucial aspect of Kubernetes storage management. This example demonstrates how to create a Persistent Volume and a Persistent Volume Claim, and how they can be bound together.

The main rules for PersistentVolume (PV) and PersistentVolumeClaim (PVC) binding in Kubernetes are:


- storageClassName must match:
    - The storageClassName of the PV and PVC must be the same (including both being empty strings for manual binding).
- Capacity must be sufficient:
    - The PV must have at least as much storage as requested by the PVC.
- Access modes must be compatible:
    - The PV must support all access modes requested by the PVC.
- PV must be available:
    - The PV must not already be bound to another PVC.
- Selector (if used) must match:
  - If the PVC specifies a selector, the PV’s labels must match.
- Dynamic provisioning:
  - If the PVC requests a storageClassName and no matching PV exists, Kubernetes will try to dynamically provision a PV if a StorageClass with that name exists.


## storageClassName

Apply the following YAML to create a Persistent Volume and a Persistent Volume Claim:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

To inspect the created Persistent Volume and Persistent Volume Claim, you can use the following commands:

```bash
kubectl get pv,pvc
```
```text
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/example-pv   1Gi        RWO            Retain           Available                          <unset>                          71s

NAME                                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/example-pvc   Pending                                      standard       <unset>                 17s
```

From the output, you can see that the Persistent Volume is created with a capacity of 1Gi and is available for binding. The Persistent Volume Claim is in a pending state, waiting for a suitable Persistent Volume to bind to. The reason that provided volume is not bound yet is that the `storageClassName` is not specified in the PVC, and it defaults to cluster default StorageClass (often named `standard`). If no default StorageClass is set, the PVC’s storageClassName will be `""`.

But in most cases there is a default one and thus PVC will default to `standard` StorageClass while PV will default to `""` StorageClass. As a result, the PVC will not bind to the PV because they do not match in terms of `storageClassName`.

To solve this we either need to specify the `storageClassName` in the PVC and/or in the PV. In this example, we will set the `storageClassName` in both resources to "lol".

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
  storageClassName: "lol"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: "lol"
```

And after investigating the PV and PVC again, you will see that they are now bound together:
```text
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/example-pv   1Gi        RWO            Retain           Bound    default/example-pvc   lol            <unset>                          2m49s

NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/example-pvc   Bound    example-pv   1Gi        RWO            lol            <unset>                 2s
```

The `lol` StorageClass was able to bind the Persistent Volume Claim to the Persistent Volume successfully. This is because Kubernetes allows binding PVCs to PVs with a matching `storageClassName` even if the StorageClass itself does not exist.

On the other hand a proper way would be to use one of the existing StorageClasses (especially if we would like to use dynamical provisioning, e.g. longhorn). You can check the available StorageClasses in your cluster by running:
```bash
kubectl get storageclass
```
```text
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
hostpath             rancher.io/local-path   Delete          WaitForFirstConsumer   false                  19d
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  19d
```

But if we use the `standard` StorageClass, PVC doesn't bind to the PV yet. In the previous output you can see that `VOLUMEBINDINGMODE` is set to `WaitForFirstConsumer`. This means that the Persistent Volume will not be bound until a Pod that uses the PVC is created.

```text
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/example-pv   1Gi        RWO            Retain           Available           standard       <unset>                          3m38s

NAME                                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/example-pvc   Pending                                      standard       <unset>                 3m38s
```

To make the binding happen, we need to create a Pod that uses the Persistent Volume Claim. Here is an example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
    - name: app
      image: busybox
      command: [ "sleep", "3600" ]
      volumeMounts:
        - mountPath: /data
          name: example-volume
  volumes:
    - name: example-volume
      persistentVolumeClaim:
        claimName: example-pvc
```

After applying this Pod configuration, you can check the status of the Persistent Volume Claim again:

```bash
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/example-pv   1Gi        RWO            Retain           Bound    default/example-pvc   standard       <unset>                          4m58s

NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/example-pvc   Bound    example-pv   1Gi        RWO            standard       <unset>                 4m58s
```

## Capacity

If there is not available Persistent Volume with sufficient capacity, the Persistent Volume Claim will remain in a pending state. For example, if you try to create a PVC that requests 2Gi of storage while the only available PV has only 1Gi, the PVC will not bind.

PVC can only bound to capacity of PV that is greater or equal to requested capacity.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
  storageClassName: "lol"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv5
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
  storageClassName: "lol"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: "lol"
```

```text
NAME                           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/example-pv    1Gi        RWO            Retain           Available                         lol            <unset>                          2s
persistentvolume/example-pv5   5Gi        RWO            Retain           Bound       default/example-pvc   lol            <unset>                          2s

NAME                                STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/example-pvc   Bound    example-pv5   5Gi        RWO            lol            <unset>                 2s
```

In this case, the PVC is bound to the `example-pv5` Persistent Volume because it has sufficient capacity (5Gi) to satisfy the request (1Gi). The `example-pv` Persistent Volume remains available but unbound because it does not meet the capacity requirement of the PVC.
