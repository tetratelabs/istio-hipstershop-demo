# Istio Hipstershop Demo External Workloads

Istio collect cluster informations from the Kubernetes API. It does not know about your external workloads.
By default, external requests are allowed and Istio will use it's default `Passthrough` to send the request to the IP defined in the DNS.

As the workload is unknown, we can't apply a specific configuration.

`ServiceEntry` are used to declare specific workload (internal or external) to the mesh.

## Setup a Debug pod

Install a debug pod (an Alpine image) to run some commands from inside the mesh

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: debug
  name: debug
spec:
  replicas: 1
  selector:
    matchLabels:
      app: debug
  template:
    metadata:
      labels:
        app: debug
    spec:
      containers:
      - image: alpine:latest
        command:
          - tail
          - -f
          - /dev/null
        imagePullPolicy: Always
        name: debug
      restartPolicy: Always
EOF
```

In a terminal window, connect to the pod and install `curl`:

```bash
k exec -ti -n hipstershopistio deployment/debug -- sh

apk add -u curl
```

In another terminal window, use `stern` to display the Istio's logs:

```bash 
stern -n hipstershopistio debug -c istio-proxy -s 1s
```

Now, from the `debug` pod terminal, use curl:

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/status/200
```

You should see a log in the `stern` terminal like:

```bash
"GET /status/200 HTTP/1.1" 200 - "-" "-" 0 0 61 61 "-" "curl/7.69.1" "2b083806-d7ad-98f7-8c73-df8b0c42d7ca" "httpbin.org" "52.87.75.66:80" PassthroughCluster 10.36.2.19:34374 52.87.75.66:80 10.36.2.19:34372 - allow_any
```

We see that we are using the `PassthroughCluster`, as expected from the default config.

## Declare httpbin.org in the mesh

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin.org
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: http
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS
EOF
```

```bash
"GET /status/200 HTTP/1.1" 200 - "-" "-" 0 0 65 64 "-" "curl/7.69.1" "7cf1e375-56d0-9943-b835-ba626f84f974" "httpbin.org" "52.87.75.66:80" outbound|80||httpbin.org 10.36.2.19:35786 52.5.146.135:80 10.36.2.19:34898 - default
```

We now see that we are using the `outbound|80||httpbin.org ` cluster. `httpbin.org` is now known by the mesh and we can add some specific configuration to it.

## Add a Retry policy

Before addinf the Retry Policy, let's see what happen when we query a failed server:

```bash
time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/status/502
200
real	0m 0.09s
```

and the log:
```bash
"GET /status/502 HTTP/1.1" 502 - "-" "-" 0 0 62 62 "-" "curl/7.69.1" "b998a13c-7ae0-9562-a62a-b7400682418e" "httpbin.org" "52.87.75.66:80" outbound|80||httpbin.org 10.36.2.19:37242 54.236.246.173:80 10.36.2.19:52688 - default
```

The curl process took 90ms to complete, and the request (as seen by Istio) took 62ms and gave a 502 error.

Let's apply a rule to add a retry policy of 3 attempts: 

```bash
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
EOF
```

If we do the same query:

```bash
/ # time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/status/502
502
real	0m 0.33s
```

The curl command took 330ms.

In the logs: 

```bash
"GET /status/502 HTTP/1.1" 502 URX "-" "-" 0 0 288 288 "-" "curl/7.69.1" "065aeaef-efe2-96ff-b3be-60ccff46fee6" "httpbin.org" "52.87.75.66:80" outbound|80||httpbin.org 10.36.2.19:38084 52.5.146.135:80 10.36.2.19:37194 - -
```

The request took 288ms and returned a `502 URX` code. As the Envoy's documentation: `URX: The request was rejected because the upstream retry limit (HTTP) or maximum connect attempts (TCP) was reached.`

The request is almost 3x longer than the one without retry because Istio tried to reach the server 3 times.


