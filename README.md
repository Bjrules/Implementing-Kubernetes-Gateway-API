# Implementing-Kubernetes-Gateway-API
*This repo implements the use of kgateway controller to implement Kubernetes Gateway API. kindly bear in mind that that there are other gatewayAPI controller as well for example envoyproxy*

### To use Envoy Gateway:

1. Install the CRDs and controller:
```
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.6.1 -n envoy-gateway-system --create-namespace

 kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available

```
2. Register a GatewayClass:

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
EOF

```

### Using kgateway with Envoy proxy

1. Install the custom resources of the Kubernetes Gateway API version 1.6.1.

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml

```
