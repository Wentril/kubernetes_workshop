# ExternalName service example

An `ExternalName` service in Kubernetes allows you to map a service to an external DNS name. This is useful for integrating with services outside of your Kubernetes cluster, e.g., databases, third-party APIs or services that are not managed within your cluster.

ExternalName services do not have selectors or endpoints like other service types. Instead, they simply return the DNS name specified in the `.spec.externalName` field when queried.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-external-service
spec:
  type: ExternalName
  externalName: httpbin.org
```

This configuration creates a service named `my-external-service` that resolves to `httpbin.org`. When you access `my-external-service`, Kubernetes will redirect the request to `httpbin.org`. 

To test it out in a Kubernetes cluster we need to create a pod that can access the service:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-test-pod
spec:
  containers:
    - name: multitool
      image: praqma/network-multitool:latest
      command: [ "sleep", "infinity" ]
      tty: true
```

praqma/network-multitool is a useful image that provides various network tools like `curl`, `ping`, etc.

Once the pod is running, you can exec into it and test the service:

```bash
kubectl exec -t -i pod/network-test-pod -- bash
```

To test the external service inside the pod you can use tools like `nslookup`

```bash
nslookup my-external-service
```

```text
Server:         10.96.0.10
Address:        10.96.0.10#53

my-google-service.default.svc.cluster.local     canonical name = httpbin.org.
Name:   httpbin.org
Address: 54.158.253.62
Name:   httpbin.org
Address: 34.204.251.242
Name:   httpbin.org
Address: 184.72.237.235
Name:   httpbin.org
Address: 3.226.148.114
```

or `curl` 

```bash
curl -k https://my-external-service/get  
#  -k to ignore SSL certificate issues
#  /get is an endpoint provided by httpbin.org that returns a JSON response with request details.
```
```text
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "my-external-service",
    "User-Agent": "curl/7.79.1",
    "X-Amzn-Trace-Id": "Root=1-68874ad7-6dcb236032a3e9891d7b9356"
  },
  "origin": "37.48.24.248",
  "url": "https://my-external-service/get"
}
```
