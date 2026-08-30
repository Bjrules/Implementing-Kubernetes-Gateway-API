# Implementing-Kubernetes-Gateway-API

## The mental model shift first

| Ingress API | Gateway API |
|---|---|
| One resource type (`Ingress`) | Split into 3 roles: `GatewayClass`, `Gateway`, `HTTPRoute` |
| Behavior controlled via **annotations** (controller-specific, non-portable) | Behavior controlled via **structured fields** in the spec itself |
| One controller usually owns everything | **Multi-tenant by design** — infra team owns the Gateway, app teams own their own HTTPRoutes |

```
GatewayClass   → "what implementation handles this" (like a StorageClass, but for networking)
Gateway        → the actual listener (IP, port, TLS cert) — usually owned by platform/infra team
HTTPRoute      → routing rules (host/path → backend service) — usually owned by app teams (you, for bankapp)
```

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
## Phase One ( Setup of Kubernetes and  Installations of the Gateway API CRDs)

> Set up kubernetes and configure it `awscli` `terraform` `kubectl` `helm` etc


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

## Phase Two (Creation of Deployments, Service, Gateway, Cluster Issuer and  HTTPRoute )

1. Deployments 
i. (mysql.yaml with the service `mysql-service`)

```yaml
---
# MySQL Deployment with Resource Requests
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "Test@123"
        - name: MYSQL_DATABASE
          value: "bankappdb"
        ports:
        - containerPort: 3306
          name: mysql
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
---
# MySQL Service
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  ports:
  - port: 3306
  selector:
    app: mysql


```



