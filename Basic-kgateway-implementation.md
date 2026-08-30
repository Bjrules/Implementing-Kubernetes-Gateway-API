Kgateway HTTP implementation

##  Setup Kubernetes and install awscli, Terraform, helm kubectl etc.


### To Use `kgateway Envoy proxy` Kubernetes CRD

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

>Also see you can [install kgateway of kubernetes cluster using ArgoCD](https://kgateway.dev/docs/envoy/latest/install/argocd/)

4. Verify that the control plane is up and running.
`kubectl get pods -n kgateway-system` or `kubectl get all -n kgateway-system`

5. Verify that the kgateway GatewayClass is created. 
`kubectl get gatewayclass kgateway` or `kubectl get gatewayclass kgateway -o yaml`

See guide for uninstallation [Click Here](https://kgateway.dev/docs/envoy/latest/operations/uninstall/)

## Configure the Gateway and HTTPRoute

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
