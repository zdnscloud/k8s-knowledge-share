# Automatic mTLS
Automatically enables mutual Transport Layer Security for most HTTP-based
communication between meshed pods by establishing and authenticating secure,
private TLS connection between Linkerd proxies.

## Implementation detail
1. During install linkerd, a truct root, certificate and private key could
be generated or specified and saved into k8s secret which could only be read
by identity componenet  
1. Each proxy in sidecar has the same trust root configured in env, it also
generate its private key, when proxy startup, it connects to identity component
identity as a CA will sign certificate for proxy. The ca will include the proxy's
service account. Proxy use certificate signing request(CSR) to get the certificate

# tap
Listen to a traffic stream for a resource 

# router
1. For each service, routes could be specified
1. Route include http path and method
1. Use CRD ServiceProfile to create the route for service
```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: webapp.default.svc.cluster.local
  namespace: linkerd
spec:
  routes:
    - name: '/books' # This is used as the value for the rt_route label
      condition:
        method: POST
        pathRegex: '/books'
    - name: '/books/{id}' # This is used as the value for the rt_route label
      condition:
        method: GET
        pathRegex: '/books/\d+'
```

