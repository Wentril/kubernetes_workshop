# Assigning Pods to Nodes

By default, the Kubernetes scheduler places pods on nodes automatically based on available resources. But there are situations where you need to influence that decision:

- A pod needs hardware that only some nodes have (e.g., GPUs, SSDs).
- A pod must run in a specific region or availability zone.
- A node is reserved for a particular team or workload type.
- You want replicas spread evenly so a single node failure does not take down all instances.

Kubernetes provides four mechanisms for this, each operating from a different angle:

| Mechanism | Direction | Approach |
|---|---|---|
| **nodeSelector** | Pod → Node | Simple label match |
| **Node Affinity** | Pod → Node | Expressive label rules (required / preferred) |
| **Taints & Tolerations** | Node → Pod | Nodes repel pods; pods opt in with tolerations |
| **Topology Spread Constraints** | Pod → Cluster | Distribute pods evenly across topology domains |

---

## 1. nodeSelector

The simplest mechanism. You label a node, then reference that label in the pod spec. The scheduler only places the pod on nodes that have a matching label.

### Hands-on

First, check the labels already on your nodes:

```bash
kubectl get nodes --show-labels
```

```text
NAME           STATUS   ROLES           AGE   VERSION   LABELS
minikube       Ready    control-plane   3d    v1.33.1   beta.kubernetes.io/arch=amd64,...,kubernetes.io/hostname=minikube,...
minikube-m02   Ready    <none>          3d    v1.33.1   beta.kubernetes.io/arch=amd64,...,kubernetes.io/hostname=minikube-m02,...
minikube-m03   Ready    <none>          3d    v1.33.1   beta.kubernetes.io/arch=amd64,...,kubernetes.io/hostname=minikube-m03,...
```

Add a custom label to `minikube-m02`:

```bash
kubectl label node minikube-m02 role=frontend
```

```text
node/minikube-m02 labeled
```

Create a pod that requires this label. See: [node_selector_pod.yaml](../examples/node_assignment/node_selector_pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-node-selector
spec:
  containers:
  - name: nginx
    image: nginx:1.29.0
  nodeSelector:
    role: frontend
```

```bash
kubectl apply -f examples/node_assignment/node_selector_pod.yaml
```

Confirm it landed on `minikube-m02`:

```bash
kubectl get pod nginx-node-selector -o wide
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-node-selector    1/1     Running   0          15s   10.244.1.8   minikube-m02
```

**What happens if no node matches?** Remove the label and try again:

```bash
kubectl label node minikube-m02 role-     # removes the label
kubectl delete pod nginx-node-selector
kubectl apply -f examples/node_assignment/node_selector_pod.yaml
kubectl get pod nginx-node-selector
```

```text
NAME                   READY   STATUS    RESTARTS   AGE
nginx-node-selector    0/1     Pending   0          30s
```

The pod stays `Pending` indefinitely because no node satisfies the selector. `kubectl describe` tells you why:

```bash
kubectl describe pod nginx-node-selector
```

```text
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  30s   default-scheduler  0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

Clean up before continuing:

```bash
kubectl delete pod nginx-node-selector
kubectl label node minikube-m02 role=frontend   # restore for later examples
```

---

## 2. Node Affinity

Node Affinity is a more expressive version of `nodeSelector`. It supports:

- **Required rules** (`requiredDuringSchedulingIgnoredDuringExecution`): the pod will not be scheduled unless the rule is satisfied — same hard constraint as `nodeSelector`.
- **Preferred rules** (`preferredDuringSchedulingIgnoredDuringExecution`): the scheduler tries to satisfy the rule but will still schedule the pod if no node matches.

The `IgnoredDuringExecution` part means that if a node's labels change after a pod is running, the pod is not evicted.

### Operators

Node affinity rules use `matchExpressions` with these operators:

| Operator | Meaning |
|---|---|
| `In` | Label value is in the list |
| `NotIn` | Label value is not in the list |
| `Exists` | Label key exists (any value) |
| `DoesNotExist` | Label key does not exist |
| `Gt` | Label value is greater than (numeric strings only) |
| `Lt` | Label value is less than (numeric strings only) |

### Hands-on

Add a second label so we have something to work with:

```bash
kubectl label node minikube-m03 role=general
```

Deploy a deployment with both a required and preferred rule. See: [node_affinity_deployment.yaml](../examples/node_assignment/node_affinity_deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-affinity
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx-affinity
  template:
    metadata:
      labels:
        app: nginx-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: role
                operator: In
                values:
                - frontend
          - weight: 20
            preference:
              matchExpressions:
              - key: role
                operator: In
                values:
                - general
      containers:
      - name: nginx
        image: nginx:1.29.0
```

The required rule (`kubernetes.io/os=linux`) is satisfied by all nodes — this just prevents accidental scheduling on Windows nodes. The scheduler then strongly prefers `role=frontend` (weight 80) and moderately prefers `role=general` (weight 20). The control-plane node (no `role` label) is the least preferred.

```bash
kubectl apply -f examples/node_assignment/node_affinity_deployment.yaml
kubectl get pods -o wide
```

```text
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-affinity-7d9f8c6b4-2fxjt   1/1     Running   0          10s   10.244.1.9    minikube-m02
nginx-affinity-7d9f8c6b4-5hkrp   1/1     Running   0          10s   10.244.1.10   minikube-m02
nginx-affinity-7d9f8c6b4-9qlzn   1/1     Running   0          10s   10.244.2.8    minikube-m03
nginx-affinity-7d9f8c6b4-xvmt2   1/1     Running   0          10s   10.244.1.11   minikube-m02
```

Most pods landed on `minikube-m02` (`role=frontend`, weight 80). Some went to `minikube-m03` (`role=general`, weight 20). The control-plane node received none because it also has the `NoSchedule` taint from our cluster setup — which brings us to the next mechanism.

Clean up:

```bash
kubectl delete deployment nginx-affinity
kubectl label node minikube-m02 role-
kubectl label node minikube-m03 role-
```

---

## 3. Taints and Tolerations

Where node affinity is a *preference* expressed by the pod, taints work in reverse: a taint on a node **repels** pods. Only pods that explicitly declare a matching **toleration** are allowed to schedule there.

Think of it as: taints say "stay away unless you have a pass", tolerations are the pass.

### Taint format

```
kubectl taint nodes <node-name> <key>=<value>:<effect>
```

There are three effects:

| Effect | Behaviour |
|---|---|
| `NoSchedule` | New pods without a matching toleration will not be scheduled on this node. Existing pods are unaffected. |
| `PreferNoSchedule` | The scheduler avoids this node for pods without a toleration, but will use it if no other option exists. |
| `NoExecute` | New pods without a toleration are not scheduled. Existing pods **without** a toleration are evicted. |

To remove a taint, append `-` to the taint key:

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>-
```

### You have already seen taints in action

When we set up the multi-node minikube cluster in the main course, we applied a taint to the control-plane node:

```bash
kubectl taint nodes minikube node-role.kubernetes.io/control-plane:NoSchedule
```

This is why all workloads in the hands-on labs landed only on `minikube-m02` and `minikube-m03` — the control-plane node repels normal pods.

You can see active taints with `kubectl describe`:

```bash
kubectl describe node minikube | grep -A5 Taints
```

```text
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

```bash
kubectl describe node minikube-m02 | grep -A5 Taints
```

```text
Taints:             <none>
```

### Hands-on

Taint `minikube-m03` to mark it as reserved for infrastructure workloads:

```bash
kubectl taint nodes minikube-m03 dedicated=infra:NoSchedule
```

```text
node/minikube-m03 tainted
```

Deploy a regular pod (no tolerations). With the control-plane taint on `minikube` and the new taint on `minikube-m03`, only `minikube-m02` is usable:

```bash
kubectl run nginx-no-toleration --image=nginx:1.29.0
kubectl get pod nginx-no-toleration -o wide
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-no-toleration    1/1     Running   0          8s    10.244.1.12   minikube-m02
```

Now create a pod with a toleration for the infra taint. See: [toleration_pod.yaml](../examples/node_assignment/toleration_pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-toleration
spec:
  containers:
  - name: nginx
    image: nginx:1.29.0
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "infra"
    effect: "NoSchedule"
```

```bash
kubectl apply -f examples/node_assignment/toleration_pod.yaml
kubectl get pod nginx-toleration -o wide
```

```text
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-toleration   1/1     Running   0          6s    10.244.2.9   minikube-m03
```

The toleration grants the pod *permission* to schedule on `minikube-m03`, but it does not *force* it there. The pod could still land on `minikube-m02` — use node affinity together with a toleration when you need to guarantee placement on a tainted node.

#### Toleration operators

A toleration with `operator: Equal` matches a specific key+value pair. You can also use `operator: Exists` to tolerate any value for a key, or to match a taint that has no value:

```yaml
# Tolerates dedicated=anything:NoSchedule
tolerations:
- key: "dedicated"
  operator: "Exists"
  effect: "NoSchedule"

# Tolerates ALL taints on the node (use with caution)
tolerations:
- operator: "Exists"
```

#### `NoExecute` and eviction

`NoExecute` goes further than `NoSchedule` — it also evicts already-running pods that lack a matching toleration.

Add a `NoExecute` taint to `minikube-m02` and watch what happens to the running pod:

```bash
kubectl taint nodes minikube-m02 test=evict:NoExecute
kubectl get pod nginx-no-toleration -w
```

```text
NAME                   READY   STATUS        RESTARTS   AGE
nginx-no-toleration    1/1     Terminating   0          3m
```

The pod is evicted immediately because it has no toleration for `test=evict:NoExecute`. Remove the taint to restore the node:

```bash
kubectl taint nodes minikube-m02 test=evict:NoExecute-
```

#### Toleration with `tolerationSeconds`

You can allow a pod to keep running on a `NoExecute`-tainted node for a grace period before eviction:

```yaml
tolerations:
- key: "test"
  operator: "Equal"
  value: "evict"
  effect: "NoExecute"
  tolerationSeconds: 60   # stay up to 60 seconds, then evict
```

This is useful for graceful draining scenarios.

#### Clean up

```bash
kubectl delete pod nginx-no-toleration nginx-toleration
kubectl taint nodes minikube-m03 dedicated=infra:NoSchedule-
```

---

## 4. Topology Spread Constraints

The mechanisms above control *which* node a pod can go to. Topology Spread Constraints control *how evenly* pods are distributed across a set of topology domains — whether that's individual nodes, availability zones, or any other grouping defined by a label.

The key motivation: if you have 6 replicas and all 6 land on the same node, a single node failure takes down your entire application. Topology Spread Constraints enforce a spread that limits the blast radius.

### Key fields

```yaml
topologySpreadConstraints:
- maxSkew: <N>
  topologyKey: <label-key>
  whenUnsatisfiable: DoNotSchedule | ScheduleAnyway
  labelSelector:
    matchLabels:
      <pod-labels>
```

- **`topologyKey`**: the node label used to define domains. `kubernetes.io/hostname` makes each node its own domain. `topology.kubernetes.io/zone` groups nodes by cloud availability zone.
- **`maxSkew`**: the maximum allowed difference in pod count between the most-loaded and least-loaded domain. `maxSkew: 1` means no domain can have more than one extra pod compared to any other.
- **`whenUnsatisfiable`**:
  - `DoNotSchedule`: treat the constraint as a hard requirement; the pod stays `Pending` if the spread cannot be satisfied.
  - `ScheduleAnyway`: treat it as a preference; schedule the pod even if the constraint cannot be met.
- **`labelSelector`**: which pods to count when calculating skew. Usually matches the deployment's own pod labels.

### Hands-on

Deploy 6 replicas with a constraint requiring even spread across nodes. See: [topology_spread_deployment.yaml](../examples/node_assignment/topology_spread_deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-spread
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nginx-spread
  template:
    metadata:
      labels:
        app: nginx-spread
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx-spread
      containers:
      - name: nginx
        image: nginx:1.29.0
```

```bash
kubectl apply -f examples/node_assignment/topology_spread_deployment.yaml
kubectl get pods -o wide
```

```text
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-spread-6d8f9c7b4-2fxjt   1/1     Running   0          12s   10.244.1.13   minikube-m02
nginx-spread-6d8f9c7b4-4hkrp   1/1     Running   0          12s   10.244.2.10   minikube-m03
nginx-spread-6d8f9c7b4-7qlzn   1/1     Running   0          12s   10.244.1.14   minikube-m02
nginx-spread-6d8f9c7b4-9vtm2   1/1     Running   0          12s   10.244.2.11   minikube-m03
nginx-spread-6d8f9c7b4-bxws5   1/1     Running   0          12s   10.244.1.15   minikube-m02
nginx-spread-6d8f9c7b4-rzpq8   1/1     Running   0          12s   10.244.2.12   minikube-m03
```

3 pods on `minikube-m02`, 3 on `minikube-m03`. The control-plane node (`minikube`) is excluded by its `NoSchedule` taint. The spread is perfectly even — skew is 0.

#### Observing the constraint in action

Scale down to 5 replicas. With 2 worker nodes and `maxSkew: 1`, the only valid distribution is 3+2:

```bash
kubectl scale deployment nginx-spread --replicas=5
kubectl get pods -o wide
```

```text
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-spread-6d8f9c7b4-2fxjt   1/1     Running   0          2m    10.244.1.13   minikube-m02
nginx-spread-6d8f9c7b4-4hkrp   1/1     Running   0          2m    10.244.2.10   minikube-m03
nginx-spread-6d8f9c7b4-7qlzn   1/1     Running   0          2m    10.244.1.14   minikube-m02
nginx-spread-6d8f9c7b4-9vtm2   1/1     Running   0          2m    10.244.2.11   minikube-m03
nginx-spread-6d8f9c7b4-bxws5   1/1     Running   0          2m    10.244.1.15   minikube-m02
```

3 on `minikube-m02`, 2 on `minikube-m03`. Skew = 1, which satisfies `maxSkew: 1`.

#### Simulating availability zones

In a real cloud cluster, nodes have a `topology.kubernetes.io/zone` label (e.g., `eu-west-1a`). You can simulate this in minikube by adding the label manually:

```bash
kubectl label node minikube-m02 topology.kubernetes.io/zone=zone-a
kubectl label node minikube-m03 topology.kubernetes.io/zone=zone-b
```

Then add a second constraint to the deployment that spreads across zones:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone       # spread across zones
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: nginx-spread
- maxSkew: 1
  topologyKey: kubernetes.io/hostname            # also spread within each zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: nginx-spread
```

The first constraint (hard) ensures equal distribution across zones. The second (soft) tries to avoid stacking multiple pods on a single node within the same zone. This is the pattern used in production multi-zone clusters.

#### Clean up

```bash
kubectl delete deployment nginx-spread
kubectl label node minikube-m02 topology.kubernetes.io/zone-
kubectl label node minikube-m03 topology.kubernetes.io/zone-
```

---

## Combining Mechanisms

These mechanisms are designed to work together. A common production pattern:

- **Taint** infra/GPU nodes so only dedicated workloads land there.
- **Toleration + Node Affinity** on the dedicated workload to both permit and prefer that node.
- **Topology Spread Constraints** to distribute replicas of that workload evenly across the tainted nodes.

Example pod spec combining all three:

```yaml
spec:
  tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values:
            - nvidia-tesla-v100
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: ml-training
```

---

## Further Reading

- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
