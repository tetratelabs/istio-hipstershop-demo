# Istio Hipstershop Demo Subset

Demo of a subset configuration for the `frontend` services

Ref: https://istio.io/latest/docs/tasks/traffic-management/request-routing/

## Subset for Frontend-v1 and Frontend-v2

If you go to your Hipstershop deployment using a browser, you will see the top banner changing from grey to red. This is becase we have two frontend services started, `v1` and `v2`, and the `v2` is configured with a red banner.

To simulate a canary deployment, we are going to first only send traffic to the `v1`, then we will start sending more and more requests to the `v2` until 100% of the requests.

### Create a subset for the frontend application

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: frontend-subset
spec:
  host: frontend.hipstershopistio.svc.cluster.local
  subsets:
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
EOF
```

### Create a VirtualService to shift the traffic to V1 only

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hipstershop
spec:
  hosts:
  - "hipstershop.${INGRESSIP}.sslip.io"
  gateways:
  - hipstershop-gateway
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: apiservice
        port:
          number: 8080
  - route:
    - destination:
        host: frontend
        port:
          number: 8080
        subset: v1
      weight: 100
    - destination:
        host: frontend
        port:
          number: 8080
        subset: v2
      weight: 0
EOF
```

You can reload the Hipstershop front page, you will only get the grey version now.

### Allow 25% of the requests to the V2

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hipstershop
spec:
  hosts:
  - "hipstershop.${INGRESSIP}.sslip.io"
  gateways:
  - hipstershop-gateway
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: apiservice
        port:
          number: 8080
  - route:
    - destination:
        host: frontend
        port:
          number: 8080
        subset: v1
      weight: 75
    - destination:
        host: frontend
        port:
          number: 8080
        subset: v2
      weight: 25
EOF
```

### Exercise: Send 100% to V2

Apply a new configuration to send 100% of the traffic to the `V2` frontend.
