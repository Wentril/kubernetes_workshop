# Network policies

Default deny policies are used to restrict traffic to and from pods in a Kubernetes cluster. They can be used to enforce security policies by allowing or denying traffic based on labels, namespaces, and other criteria.

To allow traffic between two namespaces, you can create network policies that specify which pods can communicate with each other. Below are examples of network policies that allow ingress traffic from all pods in `namespace2` to all pods in `namespace1`, and egress traffic from all pods in `namespace1` to all pods in `namespace2`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-namespace1-from-namespace2-all-pods
  namespace: namespace1 # This policy applies to pods in the 'namespace1' namespace
spec:
  podSelector: {} # Selects all pods in the namespace1 namespace
  policyTypes:
    - Ingress
  ingress:
    - from:
        # This namespaceSelector allows traffic from any pod within the 'namespace2' namespace.
        - namespaceSelector:
            matchLabels:
             kubernetes.io/metadata.name: namespace2
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-from-namespace2-to-namespace1-all-pods
  namespace: namespace2 # This policy applies to pods in the 'namespace2' namespace
spec:
  podSelector: {} # Selects all pods in the namespace2 namespace
  policyTypes:
    - Egress
  egress:
    - to:
        # This namespaceSelector allows outbound traffic to any pod within the 'namespace1' namespace.
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: namespace1

```
