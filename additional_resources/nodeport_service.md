# NodePort Service Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-service # Name of your Service
  labels:
    app: nginx-web # Label for the Service itself
spec:
  selector:
    app: nginx-web # CRITICAL: Matches the 'app: nginx-web' label on your Pods
  ports:
    - protocol: TCP
      port: 80 # The port this Service itself will listen on internally
      targetPort: 80 # The port your container (Nginx) is listening on inside the Pod
      nodePort: 30080 # OPTIONAL: A specific port (30000-32767) on the Node's IP.
                      # If omitted, K8s will assign a random port in that range.
  type: NodePort # This makes it a NodePort Service
```
