# Istio Hipstershop Demo Setup

This repo contains scripts and yaml files to deploy the [Hipstershop](https://github.com/tetratelabs/microservices-demo) application in a Kubernetes cluster and test some Istio scenarios like `retries` or `canary deployment`.

This demo should work for Istio 1.6 and newer, on a Kubernetes cluster 1.15 or newer.

### Install Istioctl
We are using the `istioctl` command to install and configure Istio. 

```bash
# latest istioctl command
curl -L https://istio.io/downloadIstio | sh -

# Use this command to target a specific version
# curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.4.3 sh -

# validate the install is successful
istioctl version
```

## Install Istio
You will need a working Kubernetes cluster with `kubectl` configured to reach it.

### Install Istio Operator

```bash
istioctl operator init
```

### Install Istio

This install is using the `Demo` profile. This is NOT recommended for production or intensive workload deployment. NEVER use the `Demo` profile outside of a pure test deployment.
We are using the `zipkin` trace backend. 
Zipkin, Prometheus, Grafana and Kiali will be installed inside the `istio-system` namespace.

```bash
kubectl create namespace istio-system

kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio
spec:
  profile: demo
  values:
    gateways:
      istio-ingressgateway:
        sds:
          enabled: true
    tracing:
      enabled: true
      provider: zipkin
EOF

# wait for 2 mins to ensure setup is done
sleep 120

# We will need the IP address of the Ingress Gateway, grab it and keep it in the environment
# Run this on EKS cluster (as a DNS name is returned instead of an IP)
INGRESSHOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")
INGRESSIP=$(dig +noall +answer +short ${INGRESSHOST} | head -1)
export INGRESSIP

# Run this on any other cluster (GKE)
INGRESSIP=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
export INGRESSIP

echo $INGRESSIP
```

### Install Cert-Manager
We are going to use Cert-Manager to create self-signed certificates so we can fully test the Ingress Gateway.

```bash
kubectl create namespace cert-manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.yaml

# wait to ensure the controler is started
sleep 20

kubectl apply -n cert-manager -f - <<EOF
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
  namespace: cert-manager
spec:
  selfSigned: {}
EOF
```

### Get access from outside

We now have an Istio up and running. To get access to the dashboards like Grafana, we are going to use the Istio Ingress Gateway.
Note that will open the UI to the world WITHOUT any security.
Kiali credentials are `admin`/`admin`
```bash
# generate an "observe" certificate for all the Istio interfaces
kubectl apply -n istio-system -f - <<EOF
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: observe-cert
spec:
  secretName: observe-cert
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  organization:
  - hipstershop
  commonName: prometheus.${INGRESSIP}.sslip.io
  isCA: false
  keySize: 2048
  keyAlgorithm: rsa
  keyEncoding: pkcs1
  usages:
    - server auth
    - client auth
  dnsNames:
  - prometheus.${INGRESSIP}.sslip.io
  - grafana.${INGRESSIP}.sslip.io
  - kiali.${INGRESSIP}.sslip.io
  - tracing.${INGRESSIP}.sslip.io
  ipAddresses:
  - ${INGRESSIP}
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
EOF
```

Now we can create the `gateway` and `virtualservice` for each UI.

```bash
# Only one gateway for all the names
# this is not recommended but possible for the demo
# as we created a certificate for all of the names
kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: observe-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "prometheus.${INGRESSIP}.sslip.io"
    - "grafana.${INGRESSIP}.sslip.io"
    - "kiali.${INGRESSIP}.sslip.io"
    - "tracing.${INGRESSIP}.sslip.io"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "observe-cert" 
    hosts:
    - "prometheus.${INGRESSIP}.sslip.io"
    - "grafana.${INGRESSIP}.sslip.io"
    - "kiali.${INGRESSIP}.sslip.io"
    - "tracing.${INGRESSIP}.sslip.io"
EOF

# generate a virtualservice for prometheus UI
kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: prometheus-ui
spec:
  hosts:
  - "prometheus.${INGRESSIP}.sslip.io"
  gateways:
  - observe-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 9090
        host: prometheus
EOF

# generate a virtualservice for grafana UI
kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana-ui
spec:
  hosts:
  - "grafana.${INGRESSIP}.sslip.io"
  gateways:
  - observe-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 3000
        host: grafana
EOF

# generate a virtualservice for kiali UI
kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kiali-ui
spec:
  hosts:
  - "kiali.${INGRESSIP}.sslip.io"
  gateways:
  - observe-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 20001
        host: kiali
EOF

# generate a virtualservice for zipkin/jaeger UI
kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: tracing-ui
spec:
  hosts:
  - "tracing.${INGRESSIP}.sslip.io"
  gateways:
  - observe-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 80
        host: tracing
EOF

echo "Istio UI can be accessed at https://grafana.${INGRESSIP}.sslip.io, https://prometheus.${INGRESSIP}.sslip.io, https://kiali.${INGRESSIP}.sslip.io, https://tracing.${INGRESSIP}.sslip.io"
```

## Deploy Hipstershop Application

```bash
# Create the Namespace
kubectl create namespace hipstershopistio

# Auto-Inject Istio sidecar to each pod in this namespace
kubectl label namespace hipstershopistio istio-injection=enabled

# Deploy the Hipstershop Demo yaml
kubectl apply -f https://raw.githubusercontent.com/tetratelabs/microservices-demo/master/release/hipstershop-istio-demo.yaml
```

you will see the `LoadGenerator` pod in a `CrashLoopBackOff` state for the moment. This is normal behaviour and is because it is not yet configured with our Ingress Gateway IP. 

Now we need to configure the Istio Ingress Gateway so we can connect to the Hipstershop from outside.

```bash

##############################
# generate the SSL cert (self signed)
# The Certificate HAVE TO BE created in the same Namespace as the Gateway (istio-system)
kubectl apply -n istio-system -f - <<EOF
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: hipstershop
spec:
  secretName: hipstershop-cert
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  organization:
  - hipstershop
  commonName: hipstershop.${INGRESSIP}.sslip.io
  isCA: false
  keySize: 2048
  keyAlgorithm: rsa
  keyEncoding: pkcs1
  usages:
    - server auth
    - client auth
  dnsNames:
  - hipstershop.${INGRESSIP}.sslip.io
  ipAddresses:
  - ${INGRESSIP}
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
EOF

##############################
# apply Istio Gateway setup
kubectl apply -n hipstershopistio -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: hipstershop-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "hipstershop.${INGRESSIP}.sslip.io"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "hipstershop-cert" 
    hosts:
    - "hipstershop.${INGRESSIP}.sslip.io"
EOF

##############################
# apply Istio Gateway setup
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
EOF

echo "Hipstershop is reachable at http://hipstershop.${INGRESSIP}.sslip.io and https://hipstershop.${INGRESSIP}.sslip.io"
```

Now we need to update the `LoadGenerator` deployment to use the Ingress IP.
Use the command `kubectl edit deployment -n hipstershopistio loadgenerator` and replace `add-your-ip` by the IP defined in INGRESSIP variable.
You have to change it in two places:

```yaml
        - name: FRONTEND_ADDR
          value: https://hipstershop.101.192.33.184.sslip.io
        - name: FRONTEND_IP
          value: 101.192.33.184
```


We now have a Fully deployed working demo.

