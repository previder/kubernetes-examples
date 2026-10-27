# Previder LoadBalancer for Kubernetes

## Index

- [LoadBalancer in Kubernetes](#loadbalancer-in-kubernetes)
- [Experimental Previder LoadBalancer](#experimental-previder-loadbalancer)
    - [Usage](#usage)
- [Annotations](#annotations)
    - [kubernetes.previder.nl/loadbalancer-type](#kubernetesprevidernlloadbalancer-type)
        - [Type HTTPS](#type-https)
        - [Type IP](#type-ip)
        - [Type TCP](#type-tcp)
    - [kubernetes.previder.nl/loadbalancer-proxy-protocol](#kubernetesprevidernlloadbalancer-proxy-protocol)
        - [PROXY protocol Examples](#proxy-protocol-examples)
    - [kubernetes.previder.nl/loadbalancer-san](#kubernetesprevidernlloadbalancer-san)
    - [kubernetes.previder.nl/loadbalancer-reclaim-address](#kubernetesprevidernlloadbalancer-reclaim-address)
- [Service status](#service-status)

## LoadBalancer in Kubernetes

A Kubernetes Service of type `LoadBalancer` exposes an application externally, typically by provisioning a load balancer through the infrastructure or cloud provider.

This allows external traffic to reach services running inside the Kubernetes cluster in a controlled and efficient way.


## Experimental Previder LoadBalancer

The experimental version of the native Previder LoadBalancer is now available for use.

This release allows you to use Previder infrastructure to load balance traffic to your Kubernetes workloads. No custom NAT configuration or ingress setup is required in your own infrastructure.

### Usage

For this experimental release, you must add the following annotation to your Service resource:

```yaml
kubernetes.previder.nl/loadbalancer-experimental: "true"
```

Create a Service of type `LoadBalancer` to expose a Deployment with the following manifest:
Not all annotations are required for the service to be exposed. All annotations are described below.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    kubernetes.previder.nl/loadbalancer-experimental: "true"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: my-app
```


## Annotations

### kubernetes.previder.nl/loadbalancer-type

This annotation defines the type of load balancer that should be created.

Possible values are:

- `HTTPS`
- `IP`
- `TCP` (deprecated and will be automatically replaced by IP)

#### Type HTTPS

`HTTPS` is the default load balancer type. If the annotation is not set, it will automatically be added to the Service.

When using the `HTTPS` type, Previder handles TLS offloading and forwards the traffic to your Service. The Service receives a hostname, which is a subdomain of Previder.

As with other reverse proxies, the original client IP address is added to the `X-Forwarded-For` header.

> **This type is limited to one port.**

If more than one port is defined, the load balancer will not be created. The reason will be shown in the Service status conditions.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    kubernetes.previder.nl/loadbalancer-experimental: "true"
    kubernetes.previder.nl/loadbalancer-type: "HTTPS"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: my-app
```

```terminaloutput
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP                                                            PORT(S)        AGE
my-app  LoadBalancer   NN.NN.NN.NN     <unique-id>.prod.pke.hosted-by-previder.com   80:30579/TCP   31s
```

#### Type IP
If you want to do the TLS offloading yourself or want to expose a direct TCP or UDP service, you can set the annotation to the type `IP`.
This changes how the LoadBalancer service is exposed. The `EXTERNAL-IP` field will change to a public IP address when the load balancer is ready.
The load balancer exposes all ports of the service on this IP address. 

This way you could expose both port 80 and 443 and do TLS offloading and redirection in the cluster.

```terminaloutput
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP                                                            PORT(S)        AGE
my-app  LoadBalancer   NN.NN.NN.NN     NNN.NNN.NNN.NNN   80:30579/TCP,443:31575/TCP   31s
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    kubernetes.previder.nl/loadbalancer-experimental: "true"
    kubernetes.previder.nl/loadbalancer-type: "IP"
spec:
  type: LoadBalancer
  ports:
    - name: "port-80"
      port: 80
      targetPort: 8080
    - name: "port-443"
      port: 443
      targetPort: 8443
  selector:
    app.kubernetes.io/name: my-app
```

#### Type TCP
The TCP type has been deprecated and replaced by IP. The current version supports UDP aswell. If The type TCP is configured, it will be automatically replaces by type IP.

### kubernetes.previder.nl/loadbalancer-proxy-protocol
When using the load balancer type `TCP`, the traffic is always proxied which makes the stream lose the original client IP. To overcome this, the [PROXY protocol](https://github.com/haproxy/haproxy/blob/master/doc/proxy-protocol.txt) has been an industry standard.   
After enabling this annotation, the load balancer will use the PROXY protocol in its connections, which makes sure your pods will receive the original connection information even though the connection is proxied.

To allow the PROXY protocol, often there needs to be a trusted load balancer IP configured.
The trusted load balancer IP for Previder is `100.70.0.3`.

Important: your application, ingress controller, or proxy must explicitly support PROXY protocol. If it does not, it may see unexpected data at the start of each TCP connection and fail to handle traffic correctly.
Some examples and documentation are provided in the chapter [PROXY protocol examples](#proxy-protocol-examples).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    kubernetes.previder.nl/loadbalancer-experimental: "true"
    kubernetes.previder.nl/loadbalancer-type: "TCP"
    kubernetes.previder.nl/loadbalancer-proxy-protocol: "true"
spec:
  type: LoadBalancer
  ports:
    - name: "port-80"
      port: 80
      targetPort: 8080
    - name: "port-443"
      port: 443
      targetPort: 8443
  selector:
    app.kubernetes.io/name: my-app
```

#### PROXY protocol Examples
Below is a practical list of **Ingress / Gateway API controllers** and **proxies / reverse proxies** that can be used behind a TCP load balancer with **PROXY protocol**, with links to the relevant documentation for enabling or configuring it.

##### Kubernetes Ingress / Gateway API controllers

| Controller | Type | PROXY protocol support / notes                                                                                                                                                                                                                                                                                                                                                           | Documentation |
|---|---|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|
| **Cilium** | Ingress Controller + Gateway API | Cilium supports Kubernetes Ingress and Gateway API. Ingress/Gateway traffic is handled through Cilium’s L7 proxy; enabling Gateway API requires Cilium Gateway API support and L7 proxy enabled. PROXY protocol support can be version/config dependent, so verify against your Cilium version. This can be enabled using the Helm values or by changing the config-map `cilium-config`. | [Cilium Ingress Controller](https://docs.cilium.io/en/stable/network/servicemesh/ingress) [[1]](https://docs.cilium.io/en/stable/network/servicemesh/ingress), [Cilium Gateway API](https://docs.cilium.io/en/latest/network/servicemesh/gateway-api/gateway-api) [[2]](https://docs.cilium.io/en/latest/network/servicemesh/gateway-api/gateway-api) |
| **Calico Ingress Gateway** | Gateway API controller | Calico Ingress Gateway is based on Envoy Gateway / Gateway API. For advanced options, Calico allows using custom Envoy Gateway configuration.                                                                                                                                                                                                                                            | [Calico Ingress Gateway](https://docs.tigera.io/calico-cloud/networking/ingress-gateway/about-calico-ingress-gateway) [[3]](https://docs.tigera.io/calico-cloud/networking/ingress-gateway/about-calico-ingress-gateway), [Customize Calico Ingress Gateway](https://docs.tigera.io/calico/latest/networking/ingress-gateway/customize-ingress-gateway) [[4]](https://docs.tigera.io/calico/latest/networking/ingress-gateway/customize-ingress-gateway) |
| **Istio Ingress Gateway** | Istio Gateway + Kubernetes Gateway API support | Istio can read client IP information from PROXY protocol at the ingress gateway and expose it through headers such as `X-Forwarded-For` / `X-Envoy-External-Address`.                                                                                                                                                                                                                    | [Istio Gateway network topology / PROXY protocol](https://istio.io/latest/docs/ops/configuration/traffic-management/network-topologies) [[5]](https://istio.io/latest/docs/ops/configuration/traffic-management/network-topologies), [Istio ingress gateways](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control) [[6]](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control) |
| **Envoy Gateway** | Gateway API controller | Envoy Gateway supports configuring downstream client behavior through `ClientTrafficPolicy`; newer versions use proxy protocol settings there.                                                                                                                                                                                                                                           | [Envoy Gateway ClientTrafficPolicy](https://gateway.envoyproxy.io/latest/tasks/traffic/client-traffic-policy) [[7]](https://gateway.envoyproxy.io/latest/tasks/traffic/client-traffic-policy) |
| **HAProxy Kubernetes Ingress Controller** | Ingress Controller | HAProxy supports PROXY protocol. HAProxy Kubernetes Ingress exposes related configuration via service/backend annotations depending on use case.                                                                                                                                                                                                                                         | [HAProxy Kubernetes Ingress service annotations](https://www.haproxy.com/documentation/kubernetes-ingress/community/configuration-reference/service/) [[10]](https://www.haproxy.com/documentation/kubernetes-ingress/community/configuration-reference/service), [HAProxy Ingress configuration keys](https://haproxy-ingress.github.io/docs/configuration/keys/) [[11]](https://haproxy-ingress.github.io/docs/configuration/keys) |

##### Standalone proxies / reverse proxies

| Proxy | PROXY protocol support / notes | Documentation |
|---|---|---|
| **Envoy Proxy** | Envoy supports PROXY protocol through listener filters. This is what many Gateway API implementations use underneath. | [Envoy Gateway ClientTrafficPolicy](https://gateway.envoyproxy.io/latest/tasks/traffic/client-traffic-policy) [[7]](https://gateway.envoyproxy.io/latest/tasks/traffic/client-traffic-policy) |
| **HAProxy** | Native PROXY protocol support. HAProxy can both **accept** PROXY protocol and **send** it to backends. | [HAProxy Kubernetes Ingress service annotations](https://www.haproxy.com/documentation/kubernetes-ingress/community/configuration-reference/service/) [[10]](https://www.haproxy.com/documentation/kubernetes-ingress/community/configuration-reference/service) |
| **NGINX** | Can accept PROXY protocol on `listen` directives using `proxy_protocol`, and can use `$proxy_protocol_addr` / `$proxy_protocol_port`. | [NGINX: Accepting the PROXY Protocol](https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/) [[9]](https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol) |
| **Apache HTTP Server** | Apache can handle PROXY protocol via `mod_remoteip` in newer versions; older setups may use `mod_proxy_protocol`. | [Apache mod_proxy_protocol](https://roadrunner2.github.io/mod-proxy-protocol/) [[12]](https://roadrunner2.github.io/mod-proxy-protocol), [Apache mod_proxy](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html) [[13]](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html) |

##### Quick examples of what to look for

For **NGINX**, the important listener option usually is:

```
listen 443 ssl proxy_protocol;
```

For **Istio**, look for gateway topology / proxy protocol settings, usually around:

```yaml
gatewayTopology:
  proxyProtocol: {}
```

For **Envoy Gateway**, look for `ClientTrafficPolicy` / proxy protocol configuration.

##### Important warning

Only enable PROXY protocol if the workload explicitly supports it. If your load balancer sends PROXY protocol but the ingress controller or proxy does **not** parse it, HTTP/TLS traffic can break because the connection starts with a PROXY header before the actual request data.

### kubernetes.previder.nl/loadbalancer-reclaim-address

This annotation only applies to load balancers of type `TCP`.

By default, when a load balancer is deleted, its IP address is reserved for the same cluster for 4 hours. If a new load balancer is created within that period, it can automatically receive a previously released address.

This annotation can be used to explicitly reclaim a specific released address.

Common use cases:

- Reclaiming a specific address when multiple addresses have been released
    - If multiple addresses are available for reclamation, use this annotation to select the address that should be assigned to the Service.
- Migrating an address to another cluster within the same tenant
    - A released address remains available for 4 hours within the same Previder Portal tenant.
    - This can be used to move a load balancer address to another Kubernetes cluster.
    - Migrating an address to another tenant is not supported.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    kubernetes.previder.nl/loadbalancer-experimental: "true"
    kubernetes.previder.nl/loadbalancer-type: "TCP"
    kubernetes.previder.nl/loadbalancer-reclaim-address: "NNN.NNN.NNN.NNN"
spec:
  type: LoadBalancer
  ports:
    - name: "port-80"
      port: 80
      targetPort: 8080
    - name: "port-443"
      port: 443
      targetPort: 8443
  selector:
    app.kubernetes.io/name: my-app
```

### kubernetes.previder.nl/loadbalancer-san
Using this option, a comma-separated list can be provided as extra SANs to become available. 

During the experimental phase of the load balancer service, this option is not available.

## Service status

After a `LoadBalancer` Service is created, the Previder LoadBalancer controller updates the Service status with conditions that describe the current state of the external load balancer.

These conditions can be inspected with:
``` bash
kubectl describe service <service-name>
```
or:
``` bash
kubectl get service <service-name> -o yaml
```
### Load balancer conditions

The controller uses the following condition types:

| Condition type | Description |
|---|---|
| `PreviderLoadBalancerId` | Indicates whether the Service has been linked to a load balancer resource in the Previder platform. |
| `PreviderLoadBalancerStatus` | Indicates the current provisioning or readiness state of the load balancer. |

The `PreviderLoadBalancerId` condition is used to expose the Previder Portal identifier of the created load balancer.

Example:
``` yaml
status:
  conditions:
  - type: PreviderLoadBalancerId
    status: "True"
    reason: LoadBalancerId
    message: The Previder Portal id is <load-balancer-id>
```
The `PreviderLoadBalancerStatus` condition describes the current state of the load balancer.

Possible reasons include:

| Reason | Description |
|---|---|
| `LoadBalancerProvisioning` | The load balancer is being created or updated. |
| `LoadBalancerReady` | The load balancer has been provisioned successfully and is ready to handle traffic. |
| `LoadBalancerProvisioningFailed` | The load balancer could not be created or updated successfully. |
| `LoadBalancerNotFound` | The expected load balancer could not be found. |
| `LoadbalancerInvalidPorts` | The configured Service ports are invalid for the selected load balancer type. |
| `LoadBalancerInvalidType` | The configured load balancer type is invalid. |
| `LoadBalancerInvalidProtocol` | The configured protocol is invalid or unsupported. |

Example ready status:
``` yaml
status:
  conditions:
  - type: PreviderLoadBalancerStatus
    status: "True"
    reason: LoadBalancerReady
    message: The load balancer is ready
```
Example provisioning status:
``` yaml
status:
  conditions:
  - type: PreviderLoadBalancerStatus
    status: "False"
    reason: LoadBalancerProvisioning
    message: The load balancer is being provisioned
```
Example not found status:
``` yaml
status:
  conditions:
  - type: PreviderLoadBalancerStatus
    status: "False"
    reason: LoadBalancerNotFound
    message: The load balancer could not be found
```
### External address status

The external address assigned to the Service is shown in `status.loadBalancer.ingress`.

For an `HTTPS` load balancer, the external address is usually a hostname:
``` yaml
status:
  loadBalancer:
  ingress:
  - hostname: <unique-id>.prod.pke.hosted-by-previder.com
```
For a `TCP` load balancer, the external address is an IP address. The IP status type is `Proxy`, because traffic is proxied through the Previder load balancer before it reaches the Service.

Example TCP status:
``` yaml
status:
  loadBalancer:
    ingress:
    - ip: <public-ip-address>
      ipMode: Proxy
```
When `ipMode` is `Proxy`, the load balancer does not preserve the original TCP source address by default. If the original client IP is required, enable PROXY protocol and make sure the receiving workload, ingress controller, or proxy is configured to accept it.

## Debugging

If a load balancer is not being deployed or does not become ready, first check the Service status conditions. In most cases, the reason for the issue is reported directly in the Service status.

You can inspect the Service with:
``` bash
kubectl describe service <service-name> -n <namespace>
```
or view the full Service output with:
``` bash
kubectl get service <service-name> -n <namespace> -o yaml
```
Check the `status.conditions` field for messages related to provisioning, invalid configuration, unsupported protocols, or missing load balancer resources.

### Cloud provider logs

If the Service status conditions do not provide enough information, you can inspect the logs of the Previder cloud provider.

The Previder cloud provider runs as a DaemonSet with the label:
``` yaml
app: cloud-provider-previder
```
You can view the logs with:
``` bash 
kubectl logs -n kube-system -l app=cloud-provider-previder
```
If there are multiple pods, you can include `--all-containers` and `--prefix` to make the output easier to read:
``` bash
kubectl logs -n kube-system -l app=cloud-provider-previder --all-containers --prefix
```
To follow the logs while troubleshooting, use:
``` bash
kubectl logs -n kube-system -l app=cloud-provider-previder --all-containers --prefix -f
```

### Contacting support

If issues continue to occur, please contact Previder support and include the relevant troubleshooting information.

Preferably include:

- the cluster name
- the Service name
- the Service namespace
- the output of the Service

You can collect the Service output with:
``` bash
kubectl get service <service-name> -n <namespace> -o yaml
```
Including this information helps Previder support investigate the issue more quickly.
