# Assigning Pods to specific Nodes in Kubernetes

To assign a Pod to a specific Node in Kubernetes, you can use node labels and node selectors. Here’s how you can do it:

```bash
kubectl label nodes minikube-m03 app=iwantthisnode
```

Then, you can create a Pod manifest that specifies the node selector:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-assigned-nginx-pod
spec:
  containers:
  - name: node-assigned-nginx
    image: nginx
  nodeSelector:
    app: iwantthisnode
```

See: 
- https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

TODO extend with taints and tolerations
