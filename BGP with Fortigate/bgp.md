# BGP Setup with FortiGate / Managed Firewall

In this example, we will set up a BGP connection between a Kubernetes cluster and a FortiGate Managed Firewall.

## FortiGate Setup

When logged into your FortiGate instance, if you cannot view the `Network > BGP` menu tree, go to `System > Feature Visibility` and enable `Advanced Routing` in the `Core Features` column.

This example uses `iBGP`. When using multiple clusters, you can use `eBGP` and assign separate `AS numbers` per cluster. This allows the configuration of `Routing maps` to ensure that one cluster cannot take over the routing of another.

### BGP Router Setup

1. Navigate to `Network > BGP`.
2. Fill in the following fields:
   - **Local AS**: `65001`
   - **Router ID**: IP address of the FortiGate (e.g., `192.168.1.1`)
3. Click `Create New` in the **Neighbor Groups** section:
   - **Name**: `BGP-NEIGHBOR-GROUP-01`
   - **Remote AS**: `65001`
   - **Interface**: Interface connected to the cluster
   - **Activate IPv4**: `yes`
   - Under IPv4 Filtering, enable:
      - `Soft reconfiguration`
      - `Capability: graceful restart`
      - `Capability: route refresh`
4. Click `Create New` in the **Neighbor Ranges** section:
   - **Prefix**: IP range of the cluster (e.g., `192.168.1.0/24`)
   - **Neighbor group**: `BGP-NEIGHBOR-GROUP-01`
   - **Max neighbor number**: `0`
5. Under the **Networks** section, enter the BGP range where advertised routes are allowed. This must be a different subnet than the cluster:
   - `10.255.0.0 255.255.255.0`
6. Check the **Graceful Restart** checkbox.
7. Expand the **Advanced Options** section and set:
   - **Keepalive**: `5`
   - **Holdtime**: `15`
8. Expand the **Best Path Selection** section and check:
   - `IBGP multi path`

After applying these settings, the FortiGate is ready to receive peer requests from the subnet.

---

## CNI Setup

### Cilium

In this example, we configure Cilium to use BGP to peer with the FortiGate. From Cilium 1.16 and up, a new method of configuring BGP has become available, marking the old way as legacy.

This example shows both the legacy configuration and the new method of configuring BGP.

By default, Cilium does not act as a BGP control plane. Edit the Cilium config and add the following line:

```shell
kubectl -n kube-system edit configmap cilium-config

```
`enable-bgp-control-plane: "true"`

After updating the config map, Cilium must be restarted:
```shell
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart daemonset/cilium
```

The Cilium Operator will install the BGP CRDs in the cluster. You can verify this with:

```shell
kubectl get crd | grep cilium
```
You should see:
```text
ciliumbgppeeringpolicies.cilium.io
``` 
In versions 1.16 and up, you should also see:
```text
ciliumbgpclusterconfigs.cilium.io
ciliumbgppeerconfigs.cilium.io
ciliumbgpadvertisements.cilium.io
```

Check that Cilium has restarted (this may take longer on larger clusters):
```shell
kubectl -n kube-system get po | grep "^cilium"
```

##### Legacy
The [legacy method](https://docs.cilium.io/en/latest/network/bgp-control-plane/bgp-control-plane-v1/) uses the single CRD called `CiliumBGPPeeringPolicy` with API version `cilium.io/v2alpha1`.
Apply the following manifest to the cluster.

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
  name: cilium-bgp-peering
spec:
  nodeSelector:
    matchExpressions: # <-- We will only select worker nodes and exclude control plane nodes as these are not supposed to run workloads
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  virtualRouters:
  - localASN: 65001
    neighbors:
    - peerAddress: '192.168.1.1/32' # <-- IP address of the Fortigate in the subnet 
      peerASN: 65001
    serviceAdvertisements: # <-- specify the service types to advertise
    - LoadBalancerIP # <-- default we will only advertise LoadBalancer services when they got an IP address
    serviceSelector: # <-- select Services to advertise, optional but gives more control over which services are exposed via routes
      matchLabels:
       bgp: 'true'
```

##### BGP Control Plane Resources
When using Cilium `1.16` and up, it is strongly advised to use the [new method of configuring BGP](https://docs.cilium.io/en/latest/network/bgp-control-plane/bgp-control-plane-v2/) as the legacy configuration will be discontinued in a future version.

Apply the following manifests:
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp-clusterconfig
spec:
  nodeSelector:
    matchExpressions: # <-- We will only select worker nodes and exclude control plane nodes as these are not supposed to run workloads
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  bgpInstances:
  - name: "instance-65001"
    localASN: 65001
    peers:
    - name: "peer-65001-ipv4"
      peerASN: 65001
      peerAddress: "192.168.1.1" # <-- IP address of the Fortigate in the subnet
      peerConfigRef:
        name: "cilium-peerconfig"
```
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peerconfig
spec:
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "bgp" # <-- select Services to advertise, optional but gives more control over which services are exposed via routes
```
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPAdvertisement
metadata:
  name: bgp-advertisements
  labels:
    advertise: bgp
spec:
  advertisements:
    - advertisementType: "Service" # <-- We will advertise only service of the type LoadBalancer with a set LoadBalancer IP
      service:
        addresses:
          - LoadBalancerIP
      selector: # <-- select Services to advertise, optional but gives more control over which services are exposed via routes
        matchExpressions:
          - { key: bgp, operator: In, values: [ 'true' ] }
```

### Exposing services
After the manifest has been applied to the cluster, we can check in the Fortigate if the neighbors start showing up.
Navigate to `Network > BGP` and on the right side of the screen, click the `View In Routing Monitor` button for the neighbor information.

If this list remains empty, verify the manifests, AS numbers, and neighbor ranges.

Once peers are up, you can expose a Service.

In the example we use a single nginx pod. The service will be assigned a manual address, this can also be done via IP pools like the [Cilium LoadBalander IPAM](https://docs.cilium.io/en/stable/network/lb-ipam/).
First we create a namespace and create a deployment.
### Example: NGINX

Create a namespace and a deployment:
```shell
kubectl create ns bgp-test
kubectl -n bgp-test create deployment nginx --image nginx
```
Check that the pod is running:
```shell
kubectl -n bgp-test get pod
```
Expose the deployment with a LoadBalancer service and a manually assigned address:
```shell
kubectl -n bgp-test expose --type LoadBalancer --port 80 --load-balancer-ip=10.255.0.1 deployment nginx
```
Label the service to advertise it via BGP:
```shell
kubectl -n bgp-test label service nginx bgp=true
```
The route should now appear in FortiGate under the Paths section in the BGP screen. Click View in Routing Monitor to see the details.

All worker nodes should advertise this route.

## NAT Configuration

The cluster now peers with the FortiGate and advertises routes. This section describes how to expose the service externally.

### Port Forwarding

If no free public IPs are available, use a free port instead.

1. Go to `Policy & Objects > Virtual IPs` and click `Create New`. Fill in:
   - **Name**: `VIP test`
   - **Interface**: WAN or another inbound interface
   - **External IP address/range**: A free IP address or the chosen interface IP
   - **Map to IPv4 address/range**: `10.255.0.1`
2. Enable **Port Forwarding**
3. Fill in:
   - **Protocol**: `TCP`
   - **Port Mapping Type**: `One to one`
   - **External service port**: `80`
   - **Map to IPv4 port**: `80`

### Firewall Policies

FortiGate blocks all traffic by default, so you need to allow access:

1. Go to `Policy & Objects > Firewall Policy` and click `Create New`
2. Fill in:
   - **Name**: `Allow Test VIP`
   - **Type**: `Standard`
   - **Incoming Interface**: WAN or selected interface
   - **Outgoing Interface**: Interface connected to the cluster
   - **Source**: `all`
   - **Destination**: `Virtual IP/Server -> VIP test`
   - **Service**: `HTTP` or alternative
   - **NAT**: `disable`
3. Apply the policy
4. Open a browser and go to `http://<chosen IP address>`

You should see the default NGINX test page:

**Welcome to nginx!**

