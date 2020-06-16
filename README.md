# Istio Hipstershop Demo

This repo contains scripts and yaml files to deploy the [Hipstershop](https://github.com/tetratelabs/microservices-demo) application in a Kubernetes cluster and test some Istio scenarios like `retries` or `canary deployment`.

This demo should work for Istio 1.6 and newer, on a Kubernetes cluster 1.15 or newer.

## Tools

This demo is using some specific tools.
Please ensure you have them installed:
- Stern: https://github.com/wercker/stern



## Index

1. [Setup](setup.md)
2. [Subsets](subset.md)
   Demonstrate the use of `subset` to route traffic to different versions
3. [Timeouts](timeout.md)
   Demonstrate the use of `Timeouts` and `Circuit-breakers`
4. [External](external.md)
   Demonstrate the use of `ServiceEntry` and `Retry policy`