# Ingress rules, path types, wildcards and other details

Additional details about Ingress rules, path types, wildcards, and the `ingressClassName` field.

## Ingress rules

Each HTTP rule contains the following information:
- optionally `host` - the hostname that the rule applies to
  - if not specified, the rule applies to all hosts
- `paths` - a list of paths that the rule applies to
  - each path has associated backend service (name and port)
- `backend` - described by service and port
  - HTTP(S) requests that match the rule will be forwarded to this backend service

```yaml
# .spec.rules:
    - host: nginx.example.com
      http:  # currently only HTTP rules are supported
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80  # or port name
```

## `defaultBackend`

The `defaultBackend` is the service that will handle requests that do not match any of the defined rules. 
- In case there are no rules defined, the `defaultBackend` is mandatory
- In case there is no `defaultBackend` and no rule is matched the behaviour is generally undefined and depends on specific ingress controller implementation.
  - e.g. `ingress-nginx` default backend exposes two endpoints: 
    - `/` - 404 Not Found
    - `/healthz` - 200 OK

## Path Types

There are three path types that can be used in Ingress rules:
- `Exact` - the path must match exactly, including case sensitivity
  - e.g. `/foo` will match `/foo`, but not `/Foo` or `/bar/foo`
- `Prefix` - the path must match the prefix of the request path, case sensitive
  - e.g. `/foo` will match `/foo`, `/foo/bar`, but not `/Foo` or `/bar/foo`
- `ImplementationSpecific` - the path matching is defined by the specific ingress controller implementation

for more examplse see [Path type examples](https://kubernetes.io/docs/concepts/services-networking/ingress/#examples) in Kubernetes ingress documentation

## Hostname wildcards

Hostnames can contain wildcards, which allows you to match multiple subdomains. The wildcard can only be at the beginning of the hostname. Compared to precise matches, wildcards matches compares only the suffix behind the wildcard.

Examples for `*.foo.com`:
- `bar.foo.com` - Matches based on shared suffix
- `baz.bar.foo.com` - No match, wildcard only covers a single DNS label
- `foo.com` - No match, wildcard expects one label before the suffix
- `foo.bar.com` - No match, suffix does not match


```yaml
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80
```

## `ingressClassName` vs. deprecated annotation

The `ingressClassName` field is used to specify which Ingress controller should handle the Ingress resource.

It replaces the deprecated annotation `kubernetes.io/ingress.class`. The `ingressClassName` field is preferred as it is more explicit and allows for better validation.
