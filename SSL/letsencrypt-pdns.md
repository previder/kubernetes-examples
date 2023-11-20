# LetsEncrypt automatic validation via PowerDNS
### Overview
This example will set up Cert Manager and the PDNS webhook.  
The [Previder Portal DNS API](https://portal.previder.nl/api-docs.html#/Domain%20DNS%20API) acts as a proxy to our PowerDNS servers and using any PowerDNS client, we can update the DNS for validation of the records.  
If for some reason port 80 is not available in your ingress, you can use DNS validation.

### Install Ingress Controller
Install an Ingress Controller. The controller will be installed in its own ``ingress-nginx`` namespace.  
Installing the controller can be done via Helm or manifests. We will use the manifest of the Kubernetes repo in this example, as this installs it using NodePort.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/baremetal/deploy.yaml
```

### Install Cert Manager
```shell
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.2 \
  --set installCRDs=true \
  --set dns01RecursiveNameservers="80.65.96.50:53" \
  --set dns01RecursiveNameserversOnly=true
```
 
**Wait until all the Cert Manager pods are running before installing the webhook for PDNS or it will not be initialized properly.**

### Install Cert Manager PDNS Webhook
```shell
helm repo add cert-manager-webhook-pdns https://zachomedia.github.io/cert-manager-webhook-pdns
helm install --namespace cert-manager cert-manager-webhook-pdns cert-manager-webhook-pdns/cert-manager-webhook-pdns
```
The output should similar to the following
```shell
NAME: cert-manager-webhook-pdns
LAST DEPLOYED: <date>
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

### Create token for DNS
To use the PDNS webhook we will need a token to authenticate from this addon to the Previder Portal.
Go to your [User Settings](https://portal.previder.nl/#/user/current/properties) in the top-right menu and choose the tab "Tokens".  
Create a new token and copy the secret.

Kubernetes uses Base64 based strings in their secrets, so we will need to encode it to base64 before applying.  

```shell
apiVersion: v1
kind: Secret
metadata:
  name: previder-portal-api-key
type: Opaque
data:
  key: "<base64 token>"
```

### Create an Issuer
The issuer will use the PDNS webhook to validate our record. Only replace your email address in the manifest to connect to our Previder Portal account.  
_Use the [letsEncrypt Staging environment](https://letsencrypt.org/docs/staging-environment/) when testing._
```shell
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: <youruser>@<yourdomain>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-production-account-key
    solvers:
      - dns01:
          webhook:
            groupName: acme.zacharyseguin.ca
            solverName: pdns
            config:
              host: https://portal.previder.nl
              apiKeySecretRef:
                name: previder-portal-api-key
                key: key
              apiKeyHeaderName: "X-Auth-Token"
              serverID: "previder"
              ttl: 300
              timeout: 30
```
After creating this issuer, it should automatically start to request and validate the certificate at LetsEncrypt. To see if it is working, check out your certificate status and logs of the cert-manager and cert-manager-webhook-pdns pods.  
_The verification process can take a few minutes, when the command above shows `Ready True`, you can continue and the certificate has been applied to the Ingress Route._
