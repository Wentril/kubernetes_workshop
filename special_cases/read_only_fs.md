# Read-Only File System Error in Kubernetes Pod

The basic symptom you will see is that the pod is in a `CrashLoopBackOff` state. This usually happens when the container tries to write to a file system that is mounted as read-only. As there is no additional information in the pod description, you will need to check the logs of the container to see the specific error message.

```text
Events:                                                                                                                                                                                  
    Type     Reason          Age                    From               Message                                                                                                             
    ----     ------          ----                   ----               -------                                                                                                             
    Normal   Scheduled       6m6s                   default-scheduler  Successfully assigned tss-namespace/nginx-pod to worker                                             
    Normal   NoVerification  6m7s                   Cosignwebhook      No signature verification performed                                                                                 
    Normal   Pulled          6m5s                   kubelet            Successfully pulled image "nginx" in 282ms (282ms including waiting). Image size: 192231825 bytes.                                                                                                                                                                              
    Normal   Pulled          6m4s                   kubelet            Successfully pulled image "nginx" in 261ms (261ms including waiting). Image size: 192231825 bytes.                                                                                                                                                                              
    Normal   Pulled          5m25s (x2 over 5m49s)  kubelet            Successfully pulled image "nginx" in 226ms (226ms including waiting). Image size: 192231825 bytes.                                                                                                                                                                              
    Normal   Pulled          4m36s                  kubelet            Successfully pulled image "nginx" in 230ms (230ms including waiting). Image size: 192231825 bytes.                                                                                                                                                                              
    Normal   Created         3m13s (x6 over 6m5s)   kubelet            Created container: nginx                                                                                            
    Normal   Started         3m13s (x6 over 6m4s)   kubelet            Started container nginx                                                                                             
    Normal   Pulled          3m13s                  kubelet            Successfully pulled image "nginx" in 280ms (280ms including waiting). Image size: 192231825 bytes.                                                                                                                                                                              
    Warning  BackOff         51s (x27 over 6m3s)    kubelet            Back-off restarting failed container nginx in pod nginx-pod_tss-namespace(6ed3cbff-78f3-4a71-99c5-53d703dc2929)         
    Normal   Pulling         25s (x7 over 6m5s)     kubelet            Pulling image "nginx"                                                                   
    Normal   Pulled          25s                    kubelet            Successfully pulled image "nginx" in 245ms (245ms including waiting). Image size: 192231825 bytes.                                                                                                                                                                              
```

Log files then unvail the reason of the failure:

```text
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration                                                                                         
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/                                                                                                                
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh                                                                                                    
10-listen-on-ipv6-by-default.sh: info: can not modify /etc/nginx/conf.d/default.conf (read-only file system?)                                                                            
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh                                                                                                            
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh                                                                                                        
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh                                                                                                        
/docker-entrypoint.sh: Configuration complete; ready for start up                                                                                                                        
2025/07/21 08:50:24 [warn] 1#1: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2                          
nginx: [warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2                                            
2025/07/21 08:50:24 [emerg] 1#1: mkdir() "/var/cache/nginx/client_temp" failed (30: Read-only file system)                                                                               
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (30: Read-only file system)   
```

How to fix it. In general you need to update the pod spec with appropriate volume mounts like this:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        readOnlyRootFilesystem: true
        # ... other securityContext settings
      volumeMounts:
        - name: nginx-cache
          mountPath: /var/cache/nginx
  volumes:
    - name: nginx-cache
      emptyDir: {}
```

Unfortunately in our example it is more complicated than that. It is not enough to just mount the `/var/cache/nginx` directory as an `emptyDir` volume. We also need to ensure that the nginx configuration file and other directories are mounted correctly, and that the security context is set to allow nginx to run without root privileges.

TODO: explain why each volume is needed

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: tss-namespace
  labels:
    app.kubernetes.io/name: nginx
spec:
  imagePullSecrets:
    - name: pull-secret
  containers:
    - name: nginx
      image: nginx
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        privileged: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
        - name: nginx-cache
          mountPath: /var/cache/nginx
        - name: nginx-logs
          mountPath: /var/log/nginx
        - name: nginx-tmp
          mountPath: /tmp
        - name: nginx-run
          mountPath: /var/run

  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config
    - name: nginx-cache
      emptyDir: { }
    - name: nginx-logs
      emptyDir: { }
    - name: nginx-tmp
      emptyDir: { }
    - name: nginx-run
      emptyDir: { }
  securityContext:
    fsGroup: 1000
    supplementalGroups:
      - 1000
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: tss-namespace
data:
  nginx.conf: |
    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        access_log  /var/log/nginx/access.log;

        sendfile        on;
        keepalive_timeout  65;

        server {
            listen       80;
            server_name  localhost;

            location / {
                root   /usr/share/nginx/html;
                index  index.html index.htm;
            }
        }
    }
```
