Kgateway implementation

## Setup Kubernetes and install awscli, Terraform, helm kubectl etc.

1.) Create  `gateway.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-http-gateway
  namespace: kgateway-system
spec:
  gatewayClassName: kgateway
  listeners:
  - protocol: HTTP
    port: 80
#    hostname: www.bbproject.space
    name: http
    allowedRoutes:
      namespaces:
        from: All
```

2.) Create HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bankapp-route
  namespace: webapps
spec:
  parentRefs:
    - name: my-http-gateway
      namespace: kgateway-system
  hostnames:
    - www.bbproject.space

  rules:
    - backendRefs:
        - name: bankapp-service
          port: 80


```
