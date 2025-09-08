## Resource Quotas

Resource quotas are a way to limit the amount of resources that can be consumed by a namespace. They help prevent one team or project from consuming all the cluster resources and ensure fair resource allocation across different namespaces.

Resource Quotas can limit two main types of resources:
- Compute resources (CPU and memory)
- Object counts (number of pods, services, etc.)

### Prerequisites

Assuming that following manifests are applied to the cluster:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: my-namespace # Specify the namespace here
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
```

Describe the namespace to see its current status:
```bash
kubectl describe namespace/my-namespace
```
```text
Name:         my-namespace
Labels:       kubernetes.io/metadata.name=my-namespace
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

### Resource Quota Example

Example of resource quota that limits the number of pods, CPU, and memory requests and limits in a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: example-resource-quota
  namespace: my-namespace
spec:
  hard:
    pods: "2"
    requests.memory: "256Mi"
    limits.memory: "512Mi"
    requests.cpu: "500m"
    limits.cpu: "1"
```

To apply the resource quota to the namespace, save the above YAML to a file named `namespace_resource_quota.yaml` and run:
```bash
kubectl apply -f examples/namespace_resource_quota.yaml
kubectl describe namespace/my-namespace
```
```text
Name:         my-namespace
Labels:       kubernetes.io/metadata.name=my-namespace
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:            example-resource-quota
  Resource         Used  Hard
  --------         ---   ---
  limits.cpu       0     1
  limits.memory    0     512Mi
  pods             1     2  # There is already one pod in the namespace
  requests.cpu     0     500m
  requests.memory  0     256Mi

No LimitRange resource.
```

We can see that the namespace now has a resource quota applied, which limits the number of pods to 2, CPU requests to 500m, CPU limits to 1, memory requests to 256Mi, and memory limits to 512Mi.

To test the resource quota, try to create another pod in the namespace:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
  namespace: my-namespace # Specify the namespace here
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
```
Once we attempt to apply this manifest we will get an error:
```text
Error from server (Forbidden): error when creating "examples/pods.yaml": pods "nginx2" is forbidden: failed quota: example-resource-quota: must specify limits.cpu for: nginx; limits.memory for: nginx; requests.cpu for: nginx; requests.memory for: nginx
```
As we can see it is not possible to create a new pod without specifying resource requests and limits, as the resource quota requires them to be defined.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
  namespace: my-namespace # Specify the namespace here
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
      resources: # Specify resource requests and limits
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```
As you can see, after applying the above manifest, the pod is created successfully:
```text
NAME     READY   STATUS    RESTARTS   AGE
nginx    1/1     Running   0          11m
nginx2   1/1     Running   0          10s
```
Describ pod to see its resource requests and limits (shortened output):
```text
Name:             nginx2
Namespace:        my-namespace
...
Containers:
  nginx:
    ...
    Limits:
      cpu:     500m
      memory:  256Mi
    Requests:
      cpu:        250m
      memory:     128Mi
...
```
Describe namespace:
```text
Name:         my-namespace
Labels:       kubernetes.io/metadata.name=my-namespace
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:            example-resource-quota
  Resource         Used   Hard
  --------         ---    ---
  limits.cpu       500m   1
  limits.memory    256Mi  512Mi
  pods             2      2
  requests.cpu     250m   500m
  requests.memory  128Mi  256Mi

No LimitRange resource.
```

Lets try to add one more pod `nginx3` to the namespace. This time we get another error, which indicates that we have reached the limit of pods in the namespace:
```text
Error from server (Forbidden): error when creating "examples/pods.yaml": pods "nginx3" is forbidden: exceeded quota: example-resource-quota, requested: pods=1, used: pods=2, limited: pods=2
```

Analogously, if we try to create a pod that exceeds the memory or CPU limits defined in the resource quota, we will receive an error indicating that the request exceeds the quota. To demonstrate this let's recreate all 3 pods (delete and apply the manifests again):
```text
Error from server (Forbidden): error when creating "examples/pods.yaml": pods "nginx3" is forbidden: exceeded quota: example-resource-quota, requested: limits.cpu=500m,limits.memory=256Mi,pods=1,requests.cpu=250m,requests.memory=128Mi, used: limits.cpu=1,limits.memory=512Mi,pods=2,requests.cpu=500m,requests.memory=256Mi, limited: limits.cpu=1,limits.memory=512Mi,pods=2,requests.cpu=500m,requests.memory=256Mi
```

This time all the quotas are exceeded simultaneously, so we get a more complex error message indicating that the request exceeds the limits for CPU, memory, and pods. This didn't happen before because we had there pod `nginx` without `resources` definition. This was possible because the resource quota was applied after the pod. Thus only number of pods limitation was hit as the other limits were not counted for the first pod (as they were not defined). The `ResourceQuota` is a preventative measure and is not applied retroactively to existing pods.

## Limit Ranges

While Resource Quotas set limits on the total resources a namespace can consume, the Limit Range ensures that every container has a default value, which is crucial for the ResourceQuota to work properly. There are fields `default` and `defaultRequest` that set default limits and requests respectively. On top of that, `max` and `min` fields can be used to set maximum and minimum limits and requests for containers in a namespace. Last but not least, `maxLimitRequestRatio` can be used to set the maximum ratio between limit and request for a resource.

### Prerequisites

Assuming that following manifests are applied to the cluster:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
```

Describe the namespace to see its current status:
```bash
kubectl describe namespace/my-namespace
```
```text
Name:         my-namespace
Labels:       kubernetes.io/metadata.name=my-namespace
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

### Limit Range Example

Example of a Limit Range that sets default, maximum CPU and memory requests and limits for containers in a namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example-limit-range
  namespace: my-namespace
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    type: Container
```

To apply the limit range to the namespace, save the above YAML to a file named `namespace_limit_range.yaml` and run:
```bash
kubectl apply -f examples/namespace_limit_range.yaml 
kubectl describe namespace/my-namespace
```
```text
Name:         my-namespace
Labels:       kubernetes.io/metadata.name=my-namespace
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type       Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---  ---  ---------------  -------------  -----------------------
 Container  cpu       -    -    250m             500m           -
 Container  memory    -    -    256Mi            512Mi          -
```

We can see that the namespace now has a limit range applied, which sets default CPU and memory requests and limits for containers in the namespace. Currently no `max`, `min` nor `maxLimitRequestRatio` is set.

To test the limit range, try to create a pod in the namespace without specifying resource requests and limits:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: my-namespace # Specify the namespace here
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
```
After applying this manifest, the pod is created successfully and we can see its description (shortened output):
```text
Name:             nginx
Namespace:        my-namespace
...
Containers:
  nginx:
    ...
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:        250m
      memory:     256   
...
```
As you can see from the output, the pod has been created with the default CPU and memory requests and limits defined in the limit range, even though we didn't specify them in the pod manifest.


### Specifying Request and Limit directly in pod manifest

It is still possible to specify resource requests and limits directly in the pod manifest. In this case, the values specified in the pod manifest will override the default values defined in the limit range.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
  namespace: my-namespace # Specify the namespace here
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
      resources:
          requests:
            cpu: 700m
```

Unfortunately, this will result in an error:
```text
The Pod "nginx2" is invalid: spec.containers[0].resources.requests: Invalid value: "700m": must be less than or equal to cpu limit of 500m
```

This error occurs because the request for CPU exceeds the limit defined in the default limit value set in the `LimitRange` resource. We can fix this by either increasing the limit in the `LimitRange`, by reducing the request in the pod manifest or overriding the cpu limit in the pod with a higher value.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
  namespace: my-namespace # Specify the namespace here
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
      resources:
        requests:
          cpu: 700m
        limits:
          cpu: 1000m
```
When you reply the above manifest, the pod is created successfully and we can see its description (shortened output):
```text
Name:             nginx2
Namespace:        my-namespace
...
Containers:
  nginx:
    ...
    Limits:
      cpu:     1  # 1000m, overriding the default limit in the pod manifest
      memory:  512Mi
    Requests:
      cpu:        700m  # overriding the default request in the pod manifest
      memory:     256Mi
...
```

### Limit Range type

The `type` field in the `LimitRange` resource specifies the type of resource that the limits apply to. The most common types are:
- `Container`: Limits apply to containers in a pod.
- `Pod`: Limits apply to the entire pod, summing up the resources of all containers in the pod.
- `PersistentVolumeClaim`: Limits apply to persistent volume claims to enforce minimum and maximum storage request sizes for PVCs

# Combining Resource Quotas and Limit Ranges

Resource Quotas and Limit Ranges can (and should) be used together to ensure that namespaces have fair resource allocation and that every container has a default value for resource requests and limits. 

When both are applied, the Limit Range ensures that every container has a default value, which is crucial for the Resource Quota to work properly. The Resource Quota then limits the total resources that can be consumed by the namespace, preventing one team or project from consuming all the cluster resources.

If we take examples of both resources in this intermezzo we can see the namespace description containing both restrictions:
```text
Name:         my-namespace
Labels:       kubernetes.io/metadata.name=my-namespace
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:            example-resource-quota
  Resource         Used  Hard
  --------         ---   ---
  limits.cpu       0     1
  limits.memory    0     512Mi
  pods             0     2
  requests.cpu     0     500m
  requests.memory  0     256Mi

Resource Limits
 Type       Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---  ---  ---------------  -------------  -----------------------
 Container  cpu       -    -    250m             500m           -
 Container  memory    -    -    256Mi            512Mi          -
```

Then when creating a new pod we can either specify resource requests and limits in the pod manifest or rely on the defaults defined in the Limit Range. In the same time a newly created pod cannot exceed the limits imposed by `ResourceQuota` resource.

Create a new pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: my-namespace # Specify the namespace here
spec:
  containers:
    - name: nginx
      image: nginx:1.29.0
```

And now describe the namespace:

```text
Name:         my-namespace
Labels:       kubernetes.io/metadata.name=my-namespace
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:            example-resource-quota
  Resource         Used   Hard
  --------         ---    ---
  limits.cpu       500m   1
  limits.memory    512Mi  512Mi
  pods             1      2
  requests.cpu     250m   500m
  requests.memory  256Mi  256Mi

Resource Limits
 Type       Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---  ---  ---------------  -------------  -----------------------
 Container  memory    -    -    256Mi            512Mi          -
 Container  cpu       -    -    250m             500m           -
```
