# Liveness, Readiness, and Startup Probes

Kubernetes probes are health checks that the kubelet runs against your containers. They are the primary mechanism through which Kubernetes knows whether to send traffic to a pod and whether to restart it.

Without probes, Kubernetes only knows a container has failed if the process exits with a non-zero code. A deadlocked application, an out-of-memory web server that still responds on port 80, or a pod that is still initializing — all look healthy to Kubernetes without probes.

## Three types of probes

| Probe | Question it answers | Action on failure |
|---|---|---|
| **Liveness** | Is this container still functioning? | Restart the container |
| **Readiness** | Is this container ready to serve traffic? | Remove from Service endpoints (no restart) |
| **Startup** | Has this container finished starting up? | Restart the container; blocks liveness and readiness until it passes |

### Liveness probe

The liveness probe answers: *is this container still alive?* A failing liveness probe tells the kubelet the container is stuck in an unrecoverable state, and it will be restarted according to the pod's `restartPolicy`.

Use liveness probes to recover from situations like:
- A deadlocked process that no longer responds
- An application in an infinite error loop that can't fix itself

**Do not** use a liveness probe to check external dependencies (databases, APIs). If your database goes down, the pod does not need to be restarted — it needs to wait. A liveness probe that fails in this scenario causes a restart loop that makes things worse.

### Readiness probe

The readiness probe answers: *is this container ready to receive requests?* A failing readiness probe causes the pod's IP to be removed from the Service's endpoint list — traffic stops being sent to it, but the container is **not** restarted. When the probe passes again, the pod is added back to the endpoint list automatically.

Use readiness probes to:
- Hold off traffic until the application has finished initializing (loaded caches, run migrations, etc.)
- Temporarily remove a pod from rotation during a transient overload or dependency outage

Unlike liveness, readiness probes *should* check short-lived dependency availability — the right behavior when a dependency is down is to stop accepting traffic, not to restart.

### Startup probe

Startup probes solve the problem of slow-starting applications. If you set `initialDelaySeconds` on a liveness probe long enough for the slowest start, you've also delayed failure detection during normal operation. The startup probe provides a separate, larger window only for the initial startup.

While the startup probe is running, liveness and readiness probes are disabled. Once the startup probe passes for the first time, it stops running and hands off to liveness and readiness.

A common pattern for an application that can take up to 5 minutes to start (e.g. a database, Elasticsearch, or any service with heavy initialization):

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30   # 30 attempts × 10 seconds = 5 minutes
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3    # only 30 seconds to detect a stuck running pod
```

## Probe mechanisms

Each probe type supports four check mechanisms:

### `httpGet`

Sends an HTTP GET request. Success = HTTP status 2xx or 3xx.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:               # optional
    - name: Custom-Header
      value: Awesome
```

Most common mechanism for web services. Requires the application to expose a dedicated health endpoint.

### `exec`

Runs a command inside the container. Success = exit code 0.

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

Useful when no HTTP endpoint is available, e.g., a background worker or a database process.

### `tcpSocket`

Attempts to open a TCP connection to the specified port. Success = connection established.

```yaml
livenessProbe:
  tcpSocket:
    port: 5432
```

Useful for TCP services (databases, message queues) where you just want to know the port is open.

### `grpc`

Calls the standard [gRPC health checking protocol](https://grpc.io/docs/guides/health-checking/). The application must implement the `grpc.health.v1.Health` service.

```yaml
livenessProbe:
  grpc:
    port: 50051
```

## Probe configuration fields

All three probe types share the same timing configuration:

| Field | Default | Meaning |
|---|---|---|
| `initialDelaySeconds` | 0 | Seconds to wait after container start before running the first probe |
| `periodSeconds` | 10 | How often to run the probe |
| `timeoutSeconds` | 1 | Seconds to wait for a probe response before counting it as failed |
| `successThreshold` | 1 | Consecutive successes required to mark the probe as passing (must be 1 for liveness and startup) |
| `failureThreshold` | 3 | Consecutive failures before taking action (restart / remove from endpoints) |

The total time before action = `failureThreshold × periodSeconds`.

---

## Hands-on

### Liveness probe: exec mechanism

This pod creates a file `/tmp/healthy` on startup and the liveness probe checks for it. Deleting the file simulates an application failure.

See: [liveness_exec_pod.yaml](../examples/probes/liveness_exec_pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
spec:
  containers:
  - name: app
    image: busybox:1.28
    command: ["sh", "-c", "touch /tmp/healthy && sleep 600"]
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

```bash
kubectl apply -f examples/probes/liveness_exec_pod.yaml
kubectl get pod liveness-exec
```

```text
NAME             READY   STATUS    RESTARTS   AGE
liveness-exec    1/1     Running   0          10s
```

Now trigger a liveness failure by deleting the file the probe checks for:

```bash
kubectl exec liveness-exec -- rm /tmp/healthy
```

Watch the pod — after `failureThreshold × periodSeconds` = 15 seconds the kubelet restarts the container:

```bash
kubectl get pod liveness-exec -w
```

```text
NAME             READY   STATUS    RESTARTS   AGE
liveness-exec    1/1     Running   0          30s
liveness-exec    1/1     Running   1          47s   # restarted
```

Inspect the events to see the probe failure clearly:

```bash
kubectl describe pod liveness-exec
```

```text
...
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Pulled     50s   kubelet            Successfully pulled image "busybox:1.28"
  Warning  Unhealthy  20s   kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    20s   kubelet            Container app failed liveness probe, will be restarted
  Normal   Started    19s   kubelet            Started container app
```

Clean up:

```bash
kubectl delete pod liveness-exec
```

### Readiness probe: httpGet with a Service

This example shows how a failing readiness probe pulls a pod out of a Service's endpoint list without restarting it. The pod runs nginx. The readiness probe does a GET on `/`. We simulate failure by removing the content nginx serves.

See: [readiness_http_pod.yaml](../examples/probes/readiness_http_pod.yaml)

```bash
kubectl apply -f examples/probes/readiness_http_pod.yaml
```

Wait for the pod to become ready:

```bash
kubectl get pod readiness-http
```

```text
NAME             READY   STATUS    RESTARTS   AGE
readiness-http   1/1     Running   0          15s
```

Check that the Service has an endpoint:

```bash
kubectl describe endpoints readiness-demo-svc
```

```text
Name:         readiness-demo-svc
Namespace:    default
Subsets:
  Addresses:          10.244.1.8       # pod is in the endpoint list
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  80    TCP
```

Now cause the readiness probe to fail by removing nginx's index file (GET / will return 403):

```bash
kubectl exec readiness-http -- rm /usr/share/nginx/html/index.html
```

Watch the pod — it stays Running but `READY` drops to `0/1`:

```bash
kubectl get pod readiness-http -w
```

```text
NAME             READY   STATUS    RESTARTS   AGE
readiness-http   1/1     Running   0          40s
readiness-http   0/1     Running   0          55s   # readiness probe failing
```

Check the endpoints again — the pod has been pulled from the Service:

```bash
kubectl describe endpoints readiness-demo-svc
```

```text
Subsets:
  Addresses:          <none>
  NotReadyAddresses:  10.244.1.8       # pod is excluded from traffic
```

Restore the file and the pod comes back into rotation without a restart:

```bash
kubectl exec readiness-http -- sh -c 'echo "OK" > /usr/share/nginx/html/index.html'
kubectl get pod readiness-http
```

```text
NAME             READY   STATUS    RESTARTS   AGE
readiness-http   1/1     Running   0          2m     # RESTARTS is still 0
```

Clean up:

```bash
kubectl delete -f examples/probes/readiness_http_pod.yaml
```

### Startup probe: slow-starting application

This pod simulates a slow-starting application — it waits 20 seconds before creating the file that marks it as ready. The startup probe allows up to 30 × 10s = 5 minutes for startup before giving up; liveness only kicks in after the startup probe passes.

See: [startup_probe_pod.yaml](../examples/probes/startup_probe_pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe
spec:
  containers:
  - name: app
    image: busybox:1.28
    command: ["sh", "-c", "sleep 20 && touch /tmp/ready && sleep 600"]
    startupProbe:
      exec:
        command: [cat, /tmp/ready]
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      exec:
        command: [cat, /tmp/ready]
      periodSeconds: 10
      failureThreshold: 3
```

```bash
kubectl apply -f examples/probes/startup_probe_pod.yaml
kubectl get pod startup-probe -w
```

```text
NAME            READY   STATUS    RESTARTS   AGE
startup-probe   0/1     Running   0          5s    # startup probe running, liveness inactive
startup-probe   0/1     Running   0          15s
startup-probe   1/1     Running   0          25s   # startup probe passed, liveness now active
```

The pod is not killed during the 20-second startup window despite the liveness probe being defined with a tight `failureThreshold: 3`. Without the startup probe, you would need to set `initialDelaySeconds: 25` on the liveness probe — but that also delays liveness detection after startup.

Clean up:

```bash
kubectl delete pod startup-probe
```

---

## Common mistakes

**Liveness probe that checks external dependencies** — if the liveness probe calls an external API or database and that service goes down, all pods get restarted simultaneously. This typically makes an outage far worse. Liveness should only check the health of the local process.

**No `initialDelaySeconds` on a slow-starting app** — the liveness probe fires immediately, the app hasn't started yet, the probe fails, the container is killed and restarted. This loop repeats indefinitely. Use a startup probe or set an appropriate `initialDelaySeconds`.

**Readiness and liveness pointing to the same endpoint doing the same check** — liveness should be a minimal "am I alive" check, readiness can be more thorough. Making them identical means a transient dependency issue restarts your pod instead of just removing it from rotation.

**`successThreshold > 1` on a liveness probe** — Kubernetes does not allow this (`successThreshold` must be 1 for liveness and startup probes). It will be rejected at apply time.

## Further reading

- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)
