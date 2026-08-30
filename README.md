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

### Installing `kgateway Envoy proxy` Kubernetes CRD

1. Install the custom resources of the Kubernetes Gateway API version 1.6.1.

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml

```
2. Apply the kgateway CRDs for the upgrade version by using Helm.
*Deploy the kgateway CRDs by using Helm. This command creates the kgateway-system namespace and creates the kgateway CRDs in the cluster.*

```
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version v2.4.3 kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds 

```
3. Install the kgateway Helm chart.

```
helm upgrade -i -n kgateway-system kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
--version v2.4.3

```
> see [The Official kgateway site for detailed installation instructions ](https://kgateway.dev/docs/envoy/latest/install/helm/)

4. Verify that the control plane is up and running.
`kubectl get pods -n kgateway-system` or `kubectl get all -n kgateway-system`

5. Verify that the kgateway GatewayClass is created. 
`kubectl get gatewayclass kgateway` or `kubectl get gatewayclass kgateway -o yaml`



