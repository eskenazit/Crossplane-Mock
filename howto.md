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

### 2-bis Kubernetes provider for Crossplane installation (Optional)

If want crossplane to manage further deployment (to discuss), we need to install a provider to do so.

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

### 3- Kgateways installation (Manual)

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


Let us pause for a second for talking about having several gateways on a K8s cluster. 

CRDs are global to the cluster, which means they are common to all instances. Here we simply installed two kgateways with the helm chart, so the two gateways are sharing the same API. In order to avoid this, we would need to take some actions that are outside the scope of this PoC.

Back to the gateways installation, we need to copy the follwing wode and save it into a **gateways.yaml** file

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
kubectl -n kgateway-system get deploy,svc,pods -l app=kgateway-proxy -o wide

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

We now can now set our local cluster to listen to ports 9000 and 9001 for our internal and external gaeways and forward it to the porper pods on suitable ports :
```powershell
kubectl port-forward deployment/kgateway-internal -n kgateway-system 9000:8080 &
kubectl port-forward deployment/kgateway-external -n kgateway-system 9001:8081 &
```

### 4-  httpbin app Installation

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

Then, we expose the app on both our gateways. Here, we chose to have the samme httpbin app served on both gateways respectively by *internal.naftiko.org* and *external.naftiko.org*. We do so by creating HTTPRoutes with the following yaml file, saved to **httpbin-routes.yaml** :

```yaml
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
        - name: httpbin
          port: 8000
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin-external
  namespace: httpbin
spec:
  parentRefs:
    - name: kgateway-external
      namespace: kgateway-system
  hostnames:
    - "external.naftiko.org"
  rules:
    - backendRefs:
        - name: httpbin
          port: 8000

```

and applied it 

```powershell
kubectl apply -f .\httpbin-routes.yaml
```



and finally you can test it with your favourite api tool, or curl :

```powershell
curl -i localhost:9000/headers -H "host: internal.naftiko.org"
curl -i localhost:9001/headers -H "host: external.naftiko.org"
```


Output/pyaload shoud look like this :

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





