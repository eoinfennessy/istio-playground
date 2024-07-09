# Zero-Downtime migration from SMCP-managed gateways to gateway injection

## Base setup

### Set up SMCP and SMMR

```shell
oc create namespace istio-system
oc create namespace httpbin
oc apply -f smcp.yaml
oc apply -f smmr.yaml
```

### Install httpbin and gateway

```shell
oc apply -n httpbin -f https://raw.githubusercontent.com/maistra/istio/releases/2.5.2/samples/httpbin/httpbin.yaml
oc apply -n httpbin -f https://raw.githubusercontent.com/maistra/istio/releases/2.5.2/samples/httpbin/httpbin-gateway.yaml
```

### Verify connectivity to LoadBalancer Service (used in method 1)

```shell
export SERVICE_HOST=$(oc -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl $SERVICE_HOST/headers
```

### Create OpenShift Route targetting gateway Service in httpbin namespace (used in method 2)

```shell
base_domain=$(oc get dnses.config.openshift.io cluster -o jsonpath='{.spec.baseDomain}')
sed "s/__BASE_DOMAIN__/$base_domain/g" route.yaml | oc apply -f -
```

### Verify connectivity to OpenShift Route (used in method 2)

```shell
export ROUTE_HOST=$(oc get route -n httpbin httpbin-gateway -o jsonpath='{.spec.host}')
curl $ROUTE_HOST/headers
```

## Method 1: Reuse k8s Service created by SMCP

### Create a new ingress gateway using gateway injection

The new pod template includes the same labels that are used as the selector for the Service targeting the current
ingress gateway

```shell
oc apply -f gateway-injection-method-1.yaml
```

### Verify load balancing (Optional)

In this config specifically, the new and old deployments use different ServiceAccounts, so we can verify that the
spiffes included in the headers alternate with each request.

```shell
for i in {1..10}; do curl $SERVICE_HOST/headers; done
```
```
{
  "headers": {
    ...
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=bf65a0d7e500bc9a8295dbfc195d3bfbed8fa9949d5994726a47aa9ef99c44b9;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  }
}
{
  "headers": {
    ...
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=daf316f5d4efa0e583b2f14dde4fb70e32366b6c5cd0edceb290f4c62a009d5f;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/default"
  }
}
...
```

### Scale down old Deployment (and scale up new Deployment)

This can be done gradually in production environments.

```shell
oc scale -n istio-system deployment/istio-ingressgateway --replicas 0
```

Verify that the service remains reachable.

```shell
curl $SERVICE_HOST/headers
```

```
{
  "headers": {
    ...
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=daf316f5d4efa0e583b2f14dde4fb70e32366b6c5cd0edceb290f4c62a009d5f;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/default"
  }
}
```

### Remove `app.kubernetes.io/managed-by` label (and optionally `ownerReferences`) from ingress gateway Service

Removing the `app.kubernetes.io/managed-by` will prevent the Service being deleted when the gateway is disabled
in the SMCP.

```shell
oc label service -n istio-system istio-ingressgateway app.kubernetes.io/managed-by-
```

Removing the owner reference will prevent the Service being garbage collected when the SMCP is deleted--Maybe this could
be useful when upgrading to OSSM 3?

```shell
oc patch service -n istio-system istio-ingressgateway --type='json' -p='[{"op": "remove", "path": "/metadata/ownerReferences"}]'
```

### Disable or remove the gateway in the SMCP config

This will delete the old gateway Deployment that was managed by the SMCP.

```shell
oc patch smcp -n istio-system basic --type='json' -p='[{"op": "replace", "path": "/spec/gateways/ingress/enabled", "value": false}]'
```

Verify connectivity one last time.

```shell
curl $SERVICE_HOST/headers
```

The ingress gateway Service has not been deleted and can now be saved to a file and managed alongside the other
gateway injection resources.

## Method 2: Use external load balancer (OpenShift Routes) for canary rollout of new gateway

### Create a new ingress gateway using gateway injection (including Service)

```shell
oc apply -f gateway-injection-method-2.yaml
```

### Add the new Service to the existing Route's `alternateBackends` and specify a `weight` for this Service

```shell
oc patch route -n httpbin httpbin-gateway --type='json' -p='[{"op": "add", "path": "/spec/alternateBackends/-", "value": {"kind": "Service", "name": "httpbin-ingress-canary", "weight": 10}}]'
```

...and modify the weight of the old Service:

```shell
oc patch route -n httpbin httpbin-gateway --type='json' -p='[{"op": "replace", "path": "/spec/to/weight", "value": 90}]'
```

### Verify load balancing

10% of traffic should now be sent to the new Service

```shell
for i in {1..20}; do curl $ROUTE_HOST/headers; done
```

### Route 100% of traffic to the new Service

```shell
oc patch route -n httpbin httpbin-gateway --type='json' -p='[{"op": "replace", "path": "/spec/alternateBackends/0/weight", 100}]'
oc patch route -n httpbin httpbin-gateway --type='json' -p='[{"op": "replace", "path": "/spec/to/weight", "value": 0}]'
```

### Update `spec.to` to use the new Service and remove the `alternateBackends` block

```shell
oc patch route -n httpbin httpbin-gateway --type='json' -p='[{"op": "replace", "path": "/spec/to", "value": {"kind": "Service", "name": "httpbin-ingress-canary", "weight": 100}}]'
oc patch route -n httpbin httpbin-gateway --type='json' -p='[{"op": "delete", "path": "/spec/alternateBackends/0"}]'
```

### Disable/remove the addtional ingress gateway from the SMCP spec

```shell
oc patch smcp -n istio-system basic --type='json' -p='[{"op": "replace", "path": "/spec/gateways/additionalIngress/http-ingress/enabled", "value": false}]'
```