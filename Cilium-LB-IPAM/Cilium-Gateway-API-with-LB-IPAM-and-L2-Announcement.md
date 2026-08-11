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
- Helm based Cilium installation

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
kubectl -n kube-system get configmap cilium-config -o yaml | grep gateway
```

Gateway API must be enabled:

```yaml
enable-gateway-api: "true"
```

If you installed Cilium using Helm:

```bash
helm get values cilium -n kube-system
```

You should have:

```yaml
gatewayAPI:
  enabled: true
```

If required:

```bash
helm upgrade cilium cilium/cilium \
-n kube-system \
--reuse-values \
--set gatewayAPI.enabled=true
```

---

# 3. Create GatewayClass

The GatewayClass tells Kubernetes which controller manages the Gateway.

Create:

`01-gateway-class.yaml`

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
kubectl apply -f 01-gateway-class.yaml
```

Verify:

```bash
kubectl get gatewayclass
```

Expected:

```
NAME      CONTROLLER
cilium    io.cilium/gateway-controller
```

---

# 4. Create Gateway

The Gateway is the external entry point.

The Gateway will automatically receive an IP address from the existing Cilium LB-IPAM pool.

Create:

`02-gateway.yaml`

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
kubectl apply -f 02-gateway.yaml
```

Check:

```bash
kubectl get gateway
```

Example:

```
NAME            ADDRESS

demo-gateway    192.168.1.240
```

This IP comes from:

```
CiliumLoadBalancerIPPool
```

and is advertised by:

```
CiliumL2AnnouncementPolicy
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

`03-hello-app-1.yaml`

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
kubectl apply -f 03-hello-app-1.yaml
```

---

## Application 2

Create:

`04-hello-app-2.yaml`

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
kubectl apply -f 04-hello-app-2.yaml
```

---

# 6. Create HTTPRoutes

Now we connect HTTP traffic to the correct application.

We will use host based routing:

```
app1.example.local
        |
        |
 hello-app-1


app2.example.local
        |
        |
 hello-app-2
```

---

## Route application 1

Create:

`05-route-app-1.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute

metadata:

  name: hello-app-1-route


spec:

  parentRefs:

  - name: demo-gateway


  hostnames:

  - "app1.example.local"


  rules:

  - backendRefs:

    - name: hello-app-1

      port: 80
```

Apply:

```bash
kubectl apply -f 05-route-app-1.yaml
```

---

## Route application 2

Create:

`06-route-app-2.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute

metadata:

  name: hello-app-2-route


spec:

  parentRefs:

  - name: demo-gateway


  hostnames:

  - "app2.example.local"


  rules:

  - backendRefs:

    - name: hello-app-2

      port: 80
```

Apply:

```bash
kubectl apply -f 06-route-app-2.yaml
```

---

# 7. Configure local DNS for testing

For a local test environment, add entries to your workstation:

Linux:

```bash
sudo nano /etc/hosts
```

Add:

```
192.168.1.240 app1.example.local
192.168.1.240 app2.example.local
```

Both applications now use the same IP address.

---

# 8. Test

Test application 1:

```bash
curl http://app1.example.local
```

Expected:

```
Hello from application 1
```

Test application 2:

```bash
curl http://app2.example.local
```

Expected:

```
Hello from application 2
```

---

# Troubleshooting

## Check Gateway status

```bash
kubectl get gateway
```

Expected:

```
NAME             ADDRESS
demo-gateway     192.168.1.240
```

---

## Check HTTPRoutes

```bash
kubectl get httproute
```

Expected:

```
hello-app-1-route
hello-app-2-route
```

---

## Check Services

```bash
kubectl get svc
```

The Gateway service should have the LoadBalancer IP.

---

# Best Practices

## Use a dedicated LoadBalancer IP range

Example:

```
DHCP range:

192.168.1.50 - 192.168.1.200


Cilium LoadBalancer pool:

192.168.1.240 - 192.168.1.250
```

Never overlap these ranges.

---

## Use DNS instead of IP addresses

Avoid:

```
http://192.168.1.240
```

Prefer:

```
https://app1.example.local
```

---

## Keep one Gateway for multiple applications

Recommended:

```
                 Gateway

                    |

        +-----------+-----------+

        |                       |

   Application 1          Application 2
```

Avoid creating a LoadBalancer IP for every application.

---

# Summary

The complete traffic flow:

```
User

 |

DNS

 |

Gateway IP

 |

Cilium L2 Announcement

 |

Cilium Gateway API

 |

HTTPRoute

 |

Kubernetes Service

 |

Pods
```

Cilium LB-IPAM provides the IP address.

Cilium L2 Announcement makes the IP reachable.

Gateway API provides application-level routing.

Together they provide a modern bare-metal Kubernetes ingress solution.
