# Cilium Gateway API with LB-IPAM and L2 Announcement

## Overview

This guide is an extension of the Cilium LoadBalancer IPAM example.

The goal is to demonstrate how to expose multiple Kubernetes applications using:

- Cilium LoadBalancer IPAM
- Cilium L2 Announcement
- Kubernetes Gateway API
- HTTPRoute resources

The example uses two `hello-kubernetes` applications behind a single Gateway IP.

The final architecture:

```
                         Client

                            |
                            |
                    192.168.1.240

                            |
                            |
              Cilium L2 Announcement

                            |
                            |
                 Cilium Gateway API

                    /                 \

                   /                   \

        hello-app-1                hello-app-2

        app1.example.local         app2.example.local

                   |                   |

                Service             Service

                   |                   |

                  Pods               Pods
```

---

# Prerequisites

This guide assumes:

- A running Kubernetes cluster
- Cilium installed
- Cilium LB-IPAM enabled
- Cilium L2 Announcement enabled

Check Cilium:

```bash
kubectl get pods -n kube-system | grep cilium
```

Example:

```
cilium-xxxxx             Running
cilium-operator-xxxxx    Running
```
Before continuing, check which version of Cilium is currently running in the cluster. The Gateway API configuration and required CRDs should be compatible with the installed Cilium version.

Run:
```bash
kubectl -n kube-system get pods -l k8s-app=cilium \
  -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'
```

Example output:
```
registry-proxy.previder.io/quay/cilium/cilium:v1.19.4@sha256:2eb67991eaa9368ba199c2fac2c573cb0ffdeb79184533344f42fc9a7ff6af3c
```

In this example, the currently running Cilium version is v1.19.4.

Important: Always check the Cilium version before proceeding. Use the documentation that corresponds to the installed Cilium version.

At the time of writing, this guide is based on Cilium v1.19.

For Cilium v1.19, see the official Cilium Gateway API documentation:

https://docs.cilium.io/en/v1.19/network/servicemesh/gateway-api/gateway-api/

---

# 1. Install Gateway API CRDs

Gateway API resources are not installed by default in Kubernetes.

For Cilium v1.19, the following Gateway API v1.4.1 CRDs are required:

- GatewayClass
- Gateway
- HTTPRoute
- ReferenceGrant
- GRPCRoute

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
```
Note: If TLS passthrough using TLSRoute is required, install the additional TLSRoute CRD from the experimental Gateway API resources.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```


Verify the Gateway API resources

```bash
kubectl api-resources | grep gateway
```

Expected:

```
gateways gateway.networking.k8s.io
httproutes gateway.networking.k8s.io
referencegrants gateway.networking.k8s.io
grpcroutes gateway.networking.k8s.io
```
If TLSRoute was installed, it should also appear in the output:

```
tlsroutes gateway.networking.k8s.io
```

---

# 2. Enable Cilium Gateway API

Check your Cilium configuration:

```bash
kubectl -n kube-system get configmap cilium-config -o yaml | grep enable-gateway-api
```

The expected output is:

```yaml
enable-gateway-api: "true"
```

If the output shows:

```yaml
enable-gateway-api: "false"
```
or the setting is not present, Gateway API is not enabled. Do not continue until this has been resolved.

---

# 3. Create GatewayClass

The GatewayClass tells Kubernetes which controller manages the Gateway.

Create:

`gateway-class.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName:
    io.cilium/gateway-controller
```

Apply:

```bash
kubectl apply -f gateway-class.yaml
```

Verify:

```bash
kubectl get gatewayclass
```

Expected output:

```
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       ...
```

The ACCEPTED status must be True before continuing.

If the status is Unknown, restart the Cilium Operator:

```bash
kubectl -n kube-system rollout restart deployment/cilium-operator
```

Wait for the rollout to complete:

```bash
kubectl -n kube-system rollout status deployment/cilium-operator
```

Then check the GatewayClass again:

```bash
kubectl get gatewayclass
```

# 4. Create Gateway

The Gateway is the external entry point.

The Gateway will automatically receive an IP address from the existing Cilium LB-IPAM pool.

Create:

`gateway.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

Apply:

```bash
kubectl apply -f gateway.yaml
```

Check:

```bash
kubectl get gateway
```

Example:

```
NAME            ADDRESS          PROGRAMMED    AGE
demo-gateway    192.168.1.240    True          ...
```

---

# 5. Deploy hello-kubernetes applications

We will deploy two applications:

- hello-app-1
- hello-app-2

Each application has its own Service.

---

## Application 1

Create:

`hello-app-1-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-app-1
  template:
    metadata:
      labels:
        app: hello-app-1
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.10
        env:
        - name: MESSAGE
          value: "Hello from application 1"
```

`hello-app-1-service.yaml`

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hello-app-1
spec:
  selector:
    app: hello-app-1
  ports:
  - port: 80
    targetPort: 8080
```

Apply:

```bash
kubectl apply -f hello-app-1-deployment.yaml
kubectl apply -f hello-app-1-service.yaml
```
---

## Application 2

Create:

`hello-app-2-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-app-2
  template:
    metadata:
      labels:
        app: hello-app-2
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.10
        env:
        - name: MESSAGE
          value: "Hello from application 2"
```
`hello-app-2-service.yaml`

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hello-app-2
spec:
  selector:
    app: hello-app-2
  ports:
  - port: 80
    targetPort: 8080
```

Apply:

```bash
kubectl apply -f hello-app-2-deployment.yaml
kubectl apply -f hello-app-2-service.yaml
```

---

# 6. Create HTTPRoutes

Now we connect HTTP traffic to the correct application.

---

## Route application 1

Create:

`route-app-1.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-app-1-route
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "app1.example.com"
  rules:
  - backendRefs:
    - name: hello-app-1
      port: 80
```

Apply:

```bash
kubectl apply -f route-app-1.yaml
```

---

## Route application 2

Create:

`route-app-2.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-app-2-route
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "app2.example.com"
  rules:
  - backendRefs:
    - name: hello-app-2
      port: 80
```

Apply:

```bash
kubectl apply -f route-app-2.yaml
```

---

# 7. Configure DNS and Test HTTPRoutes

Before testing the applications, make sure the hostnames configured in the HTTPRoute resources resolve to the external IP address of the Kubernetes cluster.

If you are using a registered domain, configure the DNS records for the hostnames to point to the external IP address of the cluster.

Make sure the required NAT rules are configured to forward the incoming traffic to the Kubernetes cluster.

Important: The hostname used to access the application must match the hostname configured in the corresponding HTTPRoute.

## Verify the HTTPRoutes

Check the status of the HTTPRoutes:

```bash
kubectl get httproute
```

Expected output:

```
NAME                HOSTNAMES              AGE
hello-app-1-route   ["app1.example.com"]   ...
hello-app-2-route   ["app2.example.com"]   ...
```

## Test the applications

Test both applications using their configured hostnames:

curl http://<application-1-hostname>
curl http://<application-2-hostname>

Each request should be routed through the Cilium Gateway to the corresponding application.
