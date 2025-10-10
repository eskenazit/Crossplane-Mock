*Note: all the following wommands were tested on powershell only. They are supposed to be standard, but you might need to adapt to your shell.*

### Prerequisites 
- kubectl
- helm
- kind

### 1- setup cluster

for now, we are aiming prototype, so kind will suffice

```powershell
 kind create cluster --name naftiko-proto
```

Thne we set our working context

```powershell
kubectl cluster-info --context kind-naftiko-proto
```

Check it works fine

```powershell
kubectl get nodes
```
You should see something close to the following:
```powershell
NAME                             STATUS   ROLES           AGE   VERSION
naftiko-proto-control-plane   Ready    control-plane   22s   v1.34.0
```

### 2- Crossplane installation

Crossplace installation is pretty straigtforward 

```powershell
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
```

You can check everything is fine

```powershell
kubectl get pods -n crossplane-system
```

You should have two pods running:
```powershell
NAME                             STATUS   ROLES           AGE   VERSION
crossplane-9d6976dc5-5wnss                 0/1     Init:0/1   0          3s 
crossplane-rbac-manager-5f56cd5cc6-f74sc   0/1     Init:0/1   0          3s
```

### 3 Kubernetes provider for Crossplane installation

If want crossplane to manage further deployment (to discuss), we need to install a providet to do so.

Copy the follwing wode and save it into a **kubernetes-provider.yaml** file

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.11.4
```

Then apply your manifest
```powershell
kubectl apply -f kubernetes-provider.yaml
```

Now we need to configure the provider to be able to interact with our cluster

```powershell
$ServiceAccount = kubectl -n crossplane-system get sa -o name | Select-String provider-kubernetes | ForEach-Object { $_.ToString().Replace('serviceaccount/', 'crossplane-system:') }
kubectl create clusterrolebinding provider-kubernetes-admin-binding --clusterrole cluster-admin --serviceaccount="$ServiceAccount"
```

Copy the follwing wode and save it into a **config-in-cluster.yaml** file
```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: in-cluster
spec:
  credentials:
    source: InjectedIdentity
```

And apply it
```powershell
kubectl apply -f config-in-cluster.yaml
```

### 4- Kgateways installation 

We want two gateways, one for internal use and one for external use.

First, we have to deploy the k8s Gateway CRDs

```powershell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```
Then deploy the kgateway iCRDs
```powershell
helm upgrade -i --create-namespace --namespace kgateway-system --version v2.1.0-main kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds --set controller.image.pullPolicy=Always
```

And install le kgateway controller

```powershell
helm upgrade -i kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway --namespace kgateway-system --version v2.0.3
```


Now let us pause a second for talking about having several gateways on a K8s cluster. CRDs are global to the cluster, which means they are common to all instances. If we were to simply install two kgateways with the helm chart, we would end with the two gateways sharing the same API. In order to avoid this, we need to adrress this at the conf level with some manual  action rather than the helm level. 

In order to do so, we first copy the follwing wode and save it into a **gateways.yaml** file

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kgateway-internal
  namespace: kgateway-system
spec:
  gatewayClassName: kgateway
  listeners:
  - name: http-internal
    protocol: HTTP
    port: 8080
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kgateway-external
  namespace: kgateway-system
spec:
  gatewayClassName: kgateway
  listeners:
  - name: http-external
    protocol: HTTP
    port: 8081
    allowedRoutes:
      namespaces:
        from: All
```

And apply it

```powershell
kubectl apply -f gateways.yaml
```

We check the pods are running fine.
```powershell
kubectl get pods -n kgateway-system
```

But if we check the Envoy proxies, we see that none are deployed yet:

```powershell
kubectl get svc -n kgateway-system -o wide

No resources found in kgateway-system namespace.
```

To solve this, we then need to force the NodePort service to serve our gateways

```powershell
helm upgrade -i kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway --namespace kgateway-system --version v2.0.3 --set kubeGateway.gatewayParameters.glooGateway.service.type=NodePort
```

We now should have Envoy proxies listening:
```powershell
kubectl get svc -n kgateway-system -o wide
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE     SELECTOR
kgateway            ClusterIP      10.96.143.106   <none>        9977/TCP         7m18s   app.kubernetes.io/instance=kgateway,app.kubernetes.io/name=kgateway,kgateway=kgateway
kgateway-external   LoadBalancer   10.96.173.6     <pending>     8081:32079/TCP   7m7s    app.kubernetes.io/instance=kgateway-external,app.kubernetes.io/name=kgateway-external,gateway.networking.k8s.io/gateway-name=kgateway-external
kgateway-internal   LoadBalancer   10.96.186.8     <pending>     8080:31862/TCP   7m7s    app.kubernetes.io/instance=kgateway-internal,app.kubernetes.io/name=kgateway-internal,gateway.networking.k8s.io/gateway-name=kgateway-internal
```

Note that you  see a third kgateway. This is not a trafic proxy gateway, this is the controller created by the Helm chart to interally expose health/metric/endpoints endpoints. You can differenciate it with its ClusterIP type instead of LoadBalancer.

We now can setup the routes for our gateways, even if nop app is deployed. Here, we chose to have a future httpbin app served on *internal.naftiko.org* and a future httpbin2 app served *external.naftiko.org*. We do so by creating HTTPRoutes with the following yaml file, saved to **httpbin-routes.yaml** :

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin-external
  namespace: httpbin2
spec:
  parentRefs:
    - name: kgateway-external
      namespace: kgateway-system
  hostnames:
    - "external.naftiko.org"
  rules:
    - backendRefs:
        - name: httpbin2
          port: 8001
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin-internal
  namespace: httpbin
spec:
  parentRefs:
    - name: kgateway-internal
      namespace: kgateway-system
  hostnames:
    - "internal.naftiko.org"
  rules:
    - backendRefs:
        - name: httpbin2
          port: 8000

```

and applied it 

```powershell
kubectl apply -f .\httpbin-routes.yaml
```

We now can now set our local cluster to listen to ports 9000 and 9001 for our internal and external gaeways and forward it to the porper pods on suitable ports :
```powershell
kubectl port-forward deployment/kgateway-internal -n kgateway-system 9000:8080 &
kubectl port-forward deployment/kgateway-external -n kgateway-system 9001:8081 &
```

### 5-  httpbin app Installation (Helm)

We are now ready to setup the final pieces of the p.o.c. httpbin.

First we create the httpbin app in the cluster: 

```powershell
kubectl apply -f https://raw.githubusercontent.com/kgateway-dev/kgateway/refs/heads/v2.0.x/examples/httpbin.yaml
```

This installed a httpbin app listeinng on the pod named httpbin on port 8000 for http and 9000 fore pure tcp

Check your pod is running:
```powershell
kubectl -n httpbin get pods
```


Since e already setup our gateway and set the port to the proper listener, we can directly check if this is working you can test it with your favourite api tool. Or curl :

```powershell
curl -i localhost:9000/headers -H "host: internal.naftiko.org"
```


Output/pyaload shoud look like this :

```
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-type: application/json; encoding=utf-8
date: Thu, 09 Oct 2025 11:47:25 GMT
content-length: 446
x-envoy-upstream-service-time: 49
server: envoy

{
  "headers": {
    "Accept": [
      "*/*"
    ],
    "Host": [
      "internal.naftiko.org"
    ],
    "User-Agent": [
      "curl/8.14.1"
    ],
    "X-Envoy-Expected-Rq-Timeout-Ms": [
      "15000"
    ],
    "X-Envoy-External-Address": [
      "127.0.0.1"
    ],
    "X-Forwarded-For": [
      "10.244.0.5"
    ],
    "X-Forwarded-Proto": [
      "http"
    ],
    "X-Request-Id": [
      "5ee8873e-148e-4175-903c-3f514c4b5635"
    ]
  }
}
```

### 5-  httpbin2 app Installation (Crossplane)

The previous httbin was a classic helm installation, but we want to try to install an app via Crosplane.

First we need to define a Schema (XRD) for this. To do his, we create CompositeResourceDefinition with crossplane by copying the following code to *xrd_httpbin2.yaml*. Note we created the simlpliest checma with no additional parameters for the sake of the example.

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: httpbins.platform.naftiko.org
spec:
  group: platform.naftiko.org
  names:
    kind: HTTPBin
    plural: httpbins
  scope: Cluster
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
```
 and applying it to the cluster
 ```powershell
  kubectl apply -f xrd_httpbin.yaml
 ```
 
 Then we need to create a composition instanciating the XRD, which will ber the actual declaration of all services. We conveerted the  https://raw.githubusercontent.com/kgateway-dev/kgateway/refs/heads/v2.0.x/examples/httpbin.yaml to a crossplane Composition. Copy and save the following to a *xr_compo_httpbin2.yaml*
 
 ```yaml
 apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: httpbin.platform.naftiko.org
spec:
  compositeTypeRef:
    apiVersion: platform.naftiko.org/v1alpha1
    kind: HTTPBin
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: ns
            base:
              apiVersion: kubernetes.crossplane.io/v1alpha2
              kind: Object
              spec:
                forProvider:
                  manifest:
                    apiVersion: v1
                    kind: Namespace
                    metadata:
                      name: httpbin2
                providerConfigRef:
                  name: in-cluster
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.namespace
                toFieldPath: spec.forProvider.manifest.metadata.name
                policy:
                  fromFieldPath: Optional

          - name: sa
            base:
              apiVersion: kubernetes.crossplane.io/v1alpha2
              kind: Object
              spec:
                forProvider:
                  manifest:
                    apiVersion: v1
                    kind: ServiceAccount
                    metadata:
                      name: httpbin2
                      namespace: httpbin2
                providerConfigRef:
                  name: in-cluster
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.namespace
                toFieldPath: spec.forProvider.manifest.metadata.namespace
                policy:
                  fromFieldPath: Optional

          - name: svc
            base:
              apiVersion: kubernetes.crossplane.io/v1alpha2
              kind: Object
              spec:
                forProvider:
                  manifest:
                    apiVersion: v1
                    kind: Service
                    metadata:
                      name: httpbin2
                      namespace: httpbin2
                      labels:
                        app: httpbin2
                        service: httpbin2
                    spec:
                      ports:
                        - name: http
                          port: 8001
                          targetPort: 8080
                        - name: tcp
                          port: 9001
                      selector:
                        app: httpbin2
                providerConfigRef:
                  name: in-cluster
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.namespace
                toFieldPath: spec.forProvider.manifest.metadata.namespace
                policy:
                  fromFieldPath: Optional

          - name: deploy
            base:
              apiVersion: kubernetes.crossplane.io/v1alpha2
              kind: Object
              spec:
                forProvider:
                  manifest:
                    apiVersion: apps/v1
                    kind: Deployment
                    metadata:
                      name: httpbin2
                      namespace: httpbin2
                    spec:
                      replicas: 1
                      selector:
                        matchLabels:
                          app: httpbin2
                          version: v1
                      template:
                        metadata:
                          labels:
                            app: httpbin2
                            version: v1
                        spec:
                          serviceAccountName: httpbin2
                          containers:
                            - name: httpbin2
                              image: docker.io/mccutchen/go-httpbin:v2.6.0
                              imagePullPolicy: IfNotPresent
                              command: ["go-httpbin"]
                              args: ["-port", "8080", "-max-duration", "600s"]
                              ports:
                                - containerPort: 8080
                            - name: curl
                              image: curlimages/curl:7.83.1
                              imagePullPolicy: IfNotPresent
                              resources:
                                requests:
                                  cpu: "100m"
                                limits:
                                  cpu: "200m"
                              command: ["tail", "-f", "/dev/null"]
                providerConfigRef:
                  name: in-cluster
 ```

and apply it to the cluster
```powershell
kubectl apply -f xr_compo_httpbin2.yaml
```

Finally we need to declare an app with an XR file. Copy the following block and save it to *xr_app_httpbin2.yaml*
```yaml
apiVersion: platform.naftiko.org/v1alpha1
kind: HTTPBin
metadata:
  name: httpbin2-sample
spec: {}
```
and apply it to the cluster
```powershell
kubectl apply -f xr_app_httpbin2.yaml
```

We already setup our gateway and set the port to the proper listener, so we can directly check if this is working with the following curl command:

```powershell
curl -i localhost:9001/headers -H "host: external.naftiko.org"
```

which should render a output lokking like this:

```
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-type: application/json; encoding=utf-8
date: Thu, 09 Oct 2025 13:00:19 GMT
content-length: 447
x-envoy-upstream-service-time: 5
server: envoy

{
  "headers": {
    "Accept": [
      "*/*"
    ],
    "Host": [
      "external.naftiko.org"
    ],
    "User-Agent": [
      "curl/8.14.1"
    ],
    "X-Envoy-Expected-Rq-Timeout-Ms": [
      "15000"
    ],
    "X-Envoy-External-Address": [
      "127.0.0.1"
    ],
    "X-Forwarded-For": [
      "10.244.0.10"
    ],
    "X-Forwarded-Proto": [
      "http"
    ],
    "X-Request-Id": [
      "9e84f504-7acd-4f52-8db5-2e47d84c2219"
    ]
  }
}
```
