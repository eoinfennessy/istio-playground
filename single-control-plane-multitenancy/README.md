An Istio configuration that splits a mesh into "Zones" to demonstate how soft tenant boundaries can be defined within a single control plane mesh

## Setup

1. Base setup
    ```shell
    kind create cluster --name zones
    
    kubectl create namespace istio-system
    helm install istio-base istio/base -n istio-system --set defaultRevision=default
    helm install istiod istio/istiod -n istio-system --set meshConfig.accessLogFile=/dev/stdout --wait
    ```

1. Create clients and servers in all namespaces.
    ```shell
    kubectl create namespace blue-a
    kubectl label namespace blue-a istio.io/rev=default
    kubectl apply -k apps -n blue-a
    
    kubectl create namespace blue-b
    kubectl label namespace blue-b istio.io/rev=default
    kubectl apply -k apps -n blue-b
    
    kubectl create namespace red-a
    kubectl label namespace red-a istio.io/rev=default
    kubectl apply -k apps -n red-a
    
    kubectl create namespace red-b
    kubectl label namespace red-b istio.io/rev=default
    kubectl apply -k apps -n red-b
    
    kubectl create namespace unzoned
    kubectl label namespace unzoned istio.io/rev=default
    kubectl apply -k apps -n unzoned
    ```

## Use Sidecars to limit upstream config to services in the same Zone

1. Check proxy config before applying sidecars.
    ```shell
    istioctl proxy-config all -n blue-a $(kubectl get pods -n blue-a -l app=sleep -o jsonpath='{.items[0].metadata.name}')
    ```

1. Apply sidecars limiting egress to namespaces within the Zone.
    ```shell
    kubectl apply -f blue-zone/sidecar.yaml -n blue-a
    kubectl apply -f blue-zone/sidecar.yaml -n blue-b
    
    kubectl apply -f red-zone/sidecar.yaml -n red-a
    kubectl apply -f red-zone/sidecar.yaml -n red-b
    ```

1. Check proxy config after applying sidecars--Notice that clusters, routes and endpoints for services outside the Zone 
are no longer present.
    ```shell
    istioctl proxy-config all -n blue-a $(kubectl get pods -n blue-a -l app=sleep -o jsonpath='{.items[0].metadata.name}')
    ```

## Limit access to allow only requests originating from within the same Zone

1. Enforce strict cluster-wide mTLS. This is required if creating AuthorizationPolicies that use `source.namespaces`.
    ```shell
    kubectl apply -f peer-authentication.yaml
    ```
    
    > Note: If PeerAuthentication with STRICT mTLS is applied and the upstream config for a client's proxy has been limited
    > (either by Sidecar egress constraints or exportTo annotations on Service), then requests fail with `upstream connect
    > error or disconnect/reset before headers. reset reason: connection termination`. I'm not sure, but I think this
    > happens because client traffic is unmatched and sent as passthrough without mTLS (as if accessing an external
    > service), but the Service it calls is expecting an mTLS connection. Maybe this means that creating the following
    > AuthoriationPolicies is not particularly useful when creating both Sidecars and exportTo annotations?

1. Apply AuhorizationPolicies in each namespace that `ALLOW` requests from namespaces within the Zone.
    ```shell
    kubectl apply -f blue-zone/authorization-policy.yaml -n blue-a
    kubectl apply -f blue-zone/authorization-policy.yaml -n blue-b
    
    kubectl apply -f red-zone/authorization-policy.yaml -n red-a
    kubectl apply -f red-zone/authorization-policy.yaml -n red-b
    ```

1. Requests from within the Zone succeed...
    ```shell
    kubectl exec -ti -n blue-a $(kubectl get pod -n blue-a -l app=sleep -o jsonpath='{.items[0].metadata.name}') -c sleep -- curl httpbin:8000/get
    ```

1. ...while requests from outside the Zone fail with `RBAC: access denied`.
    ```shell
    kubectl exec -ti -n unzoned $(kubectl get pod -n unzoned -l app=sleep -o jsonpath='{.items[0].metadata.name}') -c sleep -- curl httpbin.blue-a:8000/get
    ```

## Limit visibility of Services using exportTo annotations

1. Check proxy config for client in unzoned namespace before adding `exportTo` annotations to k8s Services.
    ```shell
    istioctl proxy-config all -n unzoned $(kubectl get pods -n unzoned -l app=sleep -o jsonpath='{.items[0].metadata.name}')
    ```

1. Add `exportTo` annotations to k8s Services.
    ```shell
    kubectl annotate service -n blue-a httpbin networking.istio.io/exportTo="blue-a,blue-b"
    kubectl annotate service -n blue-b httpbin networking.istio.io/exportTo="blue-a,blue-b"
    kubectl annotate service -n blue-a sleep networking.istio.io/exportTo="blue-a,blue-b"
    kubectl annotate service -n blue-b sleep networking.istio.io/exportTo="blue-a,blue-b"
    
    kubectl annotate service -n red-a httpbin networking.istio.io/exportTo="red-a,red-b"
    kubectl annotate service -n red-b httpbin networking.istio.io/exportTo="red-a,red-b"
    kubectl annotate service -n red-a sleep networking.istio.io/exportTo="red-a,red-b"
    kubectl annotate service -n red-b sleep networking.istio.io/exportTo="red-a,red-b"
    ```

1. Check proxy config for client in unzoned namespace after adding `exportTo` annotations to k8s Services--Notice that
clusters, routes and endpoints for the annotated Services are no longer present.
    ```shell
    istioctl proxy-config all -n unzoned $(kubectl get pods -n unzoned -l app=sleep -o jsonpath='{.items[0].metadata.name}')
    ```

## Overriding Zone constraints

- A Service in a Zone can be made visible to the rest of the mesh by removing/skipping the `exportTo` annotation
- Egress scope of a Zone can be extended by creating additional Sidecar resources that use `workloadSelectors`
- Additional ALLOW/DENY AuthorizationPolicies can be created to define more specific access controls

## Cleanup

1. Delete Sidecars
    ```shell
    kubectl delete -f blue-zone/sidecar.yaml -n blue-a
    kubectl delete -f blue-zone/sidecar.yaml -n blue-b
    
    kubectl delete -f red-zone/sidecar.yaml -n red-a
    kubectl delete -f red-zone/sidecar.yaml -n red-b
    ```

1. Remove `exportTo` annotations from k8s Services.
    ```shell
    kubectl annotate service -n blue-a httpbin networking.istio.io/exportTo-
    kubectl annotate service -n blue-b httpbin networking.istio.io/exportTo-
    kubectl annotate service -n blue-a sleep networking.istio.io/exportTo-
    kubectl annotate service -n blue-b sleep networking.istio.io/exportTo-
    
    kubectl annotate service -n red-a httpbin networking.istio.io/exportTo-
    kubectl annotate service -n red-b httpbin networking.istio.io/exportTo-
    kubectl annotate service -n red-a sleep networking.istio.io/exportTo-
    kubectl annotate service -n red-b sleep networking.istio.io/exportTo-
    ```

1. Delete AuthorizationPolicies
    ```shell
    kubectl delete -f blue-zone/authorization-policy.yaml -n blue-a
    kubectl delete -f blue-zone/authorization-policy.yaml -n blue-b
    
    kubectl delete -f red-zone/authorization-policy.yaml -n red-a
    kubectl delete -f red-zone/authorization-policy.yaml -n red-b
    ```

1. Delete PeerAuthentication
    ```shell
    kubectl delete -f peer-authentication.yaml
    ```

1. Delete cluster
    ```shell
    kind delete cluster --name zones
    ```