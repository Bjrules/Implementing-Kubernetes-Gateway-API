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

## Phase Two (Creation of Deployments, Service, Gateway, Cert-Manager, Cluster Issuer and  HTTPRoute )

1. Deployments  and service
> i. (mysql.yaml with the service `mysql-service`)

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
> ii. ( Create `green-svc.yaml` the bankapp-service for the Application)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bankapp-service
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: bankapp
    version: green 

```
> iii. Create the `green.yaml` which is the application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bankapp-green
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bankapp
      version: green
  template:
    metadata:
      labels:
        app: bankapp
        version: green
    spec:
      containers:
      - name: bankapp
        image: bjrules/bankapp:v29 # Adjust the image tag for the green version
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://mysql-service:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
        - name: SPRING_DATASOURCE_USERNAME
          value: root
        - name: SPRING_DATASOURCE_PASSWORD
          value: Test@123
        resources:
          requests:
            memory: "500Mi"
            cpu: "500m"
          limits:
            memory: "1000Mi"
            cpu: "1000m"
```

## 2. Install cert-manager and create/configure a Cluster Issuer for TLS/HTTPS

> i. Install Cert-Manager.

```bash

helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.19.2 \
    --set config.apiVersion=controller.config.cert-manager.io/v1alpha1 \
    --set config.kind=ControllerConfiguration \
    --set config.enableGatewayAPI=true \
    --set crds.enabled=true

    kubectl wait --for=condition=Ready pods --all -n cert-manager

```


> ii. Creating the `cluster-issuer.yaml`  (needed before any Certificate works) 

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: justbj@live.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: gate-https
                kind: Gateway

```

## 3. Create and Initialize Gateway and HTTPRoute and http-redirect

> i. Create Gateway `gateway-https.yaml` this is where TLS/HTTPS gets configured 

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gate-https
  namespace: webapps
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: kgateway # kgateway refers to the name of the control plane pod
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: "www.bbproject.space"
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "www.bbproject.space"
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: bbproject-space-tls
      allowedRoutes:
        namespaces:
          from: Same
    - name: http-redirect
      protocol: HTTP
      port: 80
      hostname: "www.bbprojects.space"

```

> ii. create HTTPRoute `route-https.yaml`

```yaml

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-routing
  namespace: webapps
spec:
  parentRefs:
    - name: gate-https    #the name of the Gatewaycreated earlier
      sectionName: https  # pointing to the https section in the listener
  hostnames:
    - "www.bbproject.space"
  rules:
    - backendRefs:
        - name: bankapp-service #refers to the green-svc.ymal file
          port: 80

```

> iii. Create HTTP Redirect `http-redirect.yaml`  (optional but recommended) 

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bankapp-redirect
  namespace: webapps
spec:
  parentRefs:
    - name: gate-https
      sectionName: http-redirect #refers to the https-redirect section in the Gateway recource
  hostnames:
      - "www.bbprojects.space"
  rules:                        # This rule here makes sure that http traffic seamlessly redirects to https
    - filters:
        - type: RequestRedirect
          requestRedirect:
             scheme: https
             statusCode: 301

```


