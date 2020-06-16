# Istio Hipstershop Demo Timeout

Demo Timeout settings

Ref: https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/

## Demo

For the purpose of this demo we are going to stop the `LoadGenerator` so we have less logs going through:

```bash
kubectl -n hipstershopistio  scale  --replicas=0 deployment/loadgenerator
```

In Hipstershop deployment, two versions of the `productcatalogservice` is deployed. One of them is a slow version, adding 1.5s of delay before anwering.
For the moment, 50% of the requests are normal and 50% are slow. The default timeout is 15s and does not apply.
We can check that by looking at the `istio-proxy` logs:

```bash
stern productcatalogservice -c istio-proxy -s 1s

productcatalogservice  "POST /hipstershop.ProductCatalogService/GetProduct HTTP/2" 200 - "-" "-" 17 169 0 0 "-" "grpc-go/1.26.0" "3147cd7f-7b5a-9900-b806-ea8fabfee8cc" "productcatalogservice.hipstershopistio:3550" "127.0.0.1:3550" inbound|3550|grpc-productcatalogservice|productcatalogservice.hipstershopistio.svc.cluster.local 127.0.0.1:54746 10.36.1.13:3550 10.36.1.11:35284 outbound_.3550_._.productcatalogservice.hipstershopistio.svc.cluster.local default

productcatalogservice-slow  "POST /hipstershop.ProductCatalogService/GetProduct HTTP/2" 200 - "-" "-" 17 185 1501 1500 "-" "grpc-go/1.26.0" "3816fd83-6978-9b87-9cac-357a6d32587d" "productcatalogservice.hipstershopistio:3550" "127.0.0.1:3550" inbound|3550|grpc-productcatalogservice|productcatalogservice.hipstershopistio.svc.cluster.local 127.0.0.1:54610 10.36.1.14:3550 10.36.1.11:59092 outbound_.3550_._.productcatalogservice.hipstershopistio.svc.cluster.local default
```

We see here that the first log, for `productcatalogservice`, took 0ms to answer, while `productcatalogservice-slow` tool 1500ms (check the last number of the group of 4)

In a real world, we may decide that 1.5s for an answer is too long and we need to informe the client that there is an error with his request.
To do that, we are going to add a timeout of 1s on the `productcatalogservice` requests:

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productcatalogservice
spec:
  hosts:
  - "productcatalogservice.hipstershopistio.svc.cluster.local"
  http:
  - timeout: 1s
    route:
      - destination:
          host: productcatalogservice.hipstershopistio.svc.cluster.local
        weight: 100
    # retries:
    #   attempts: 3
    #   perTryTimeout: 2s
    #   retryOn: gateway-error,connect-failure,refused-stream
EOF
```

Go ahead and reload the Hipstershop front page. You should see a page with an error 500 like `HTTP Status: 500 Internal Server Error`
If you check in the logs:

```bash
productcatalogservice-slow "POST /hipstershop.ProductCatalogService/ListProducts HTTP/2" 0 DC "-" "-" 5 0 999 - "-" "grpc-go/1.26.0" "a2b795de-9bfb-936a-bc22-229f8d51d49e" "productcatalogservice.hipstershopistio:3550" "127.0.0.1:3550" inbound|3550|grpc-productcatalogservice|productcatalogservice.hipstershopistio.svc.cluster.local 127.0.0.1:54610 10.36.1.14:3550 10.36.1.11:59092 outbound_.3550_._.productcatalogservice.hipstershopistio.svc.cluster.local default
```

The error code is `0 DC`, which means `Downstream Cluster Disconnected`.
As timeouts (and most of the traffic management options of Istio) are enforced on the client side, it's the client (Downstream) which cancelled the request after 1 second.
Let's check the Downstream, which is the `frontend` application:

```bash
stern frontend  -c istio-proxy -s 1s

frontend "POST /hipstershop.ProductCatalogService/ListProducts HTTP/2" 200 UT "-" "-" 5 0 999 - "-" "grpc-go/1.26.0" "d23e4bf1-c07c-9fe4-abbc-dc5885848ff1" "productcatalogservice.hipstershopistio:3550" "10.36.1.14:3550" outbound|3550||productcatalogservice.hipstershopistio.svc.cluster.local 10.36.1.11:59092 10.0.7.198:3550 10.36.1.11:44244 - -
frontend "GET / HTTP/1.1" 500 - "-" "-" 0 3341 1004 1004 "10.128.0.7" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.97 Safari/537.36" "dc2441fb-cb90-9483-aa18-53339fc91643" "hipstershop.104.198.31.181.sslip.io" "127.0.0.1:8080" inbound|8080|http-frontend|frontend.hipstershopistio.svc.cluster.local 127.0.0.1:37686 10.36.1.11:8080 10.128.0.7:0 outbound_.8080_.v1_.frontend.hipstershopistio.svc.cluster.local default
```

We have two logs here:
1. The call to the `productcatalogservice` returned a status `200 UT` after 999ms
2. because of the error on the previous call, the frontend returned an error `500` to the client

the `UT` code means `UT: Upstream request timeout in addition to 504 response code.` You can get all the codes at https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#configuration

# Circuit Breaker
For the moment, 50% of the requests are going to the slow backend.
If we want to stop trying to reach this backend, we have to use a `Circuit-Breaker`. By using a circuit breaker, Istio will stop sending requests to a failed endpoint.

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productcatalogservice
spec:
  host: productcatalogservice.hipstershopistio.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
        connectTimeout: 1s
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
EOF
```
Here, we define an outlier detection that will trigger after one request error and will stop Istio to sending requests to a failed endpoint for 30s. 
If the next request is also in error, Istio will wait 2x30s before sending a new request to the failed endpoint, then 3x 30s, etc.

## Cleanup
```
kubectl -n hipstershopistio delete DestinationRule productcatalogservice
kubectl -n hipstershopistio delete VirtualService  productcatalogservice
kubectl -n hipstershopistio scale  --replicas=1 deployment/loadgenerator
```