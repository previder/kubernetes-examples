# Headlamp
### Overview
This example will setup Headlamp in-cluster with service account token authentication. [Headlamp](https://headlamp.dev/) is a graphical user interface specifically tailored to simplify the management of Kubernetes clusters. 

### Documentation
- [Headlamp Official Documentation](https://headlamp.dev/docs/latest/)
- [Headlamp GitHub page](https://github.com/kubernetes-sigs/headlamp)

### Install Headlamp
#### Helm-based installation
```shell
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update

helm install headlamp headlamp/headlamp --namespace headlamp --create-namespace
```

By default, headlamp will be installed in the kube-system namespace. Set a custom namespace to keep its service account and RBAC objects grouped.

To configurate the helm release, use the [official Headlamp values.yaml file](https://github.com/kubernetes-sigs/headlamp/blob/main/charts/headlamp/values.yaml).

Save the file as values.yaml and pass it to the `helm install` or `helm upgrade` command:
```shell
helm install headlamp headlamp/headlamp \
    --namespace headlamp
    --create-namespace
    -f values.yaml
``` 

#### YAML Manifest installation
Headlamp maintains a YAML file for setting up a Headlamp deployment and service. Download the [official Headlamp YAML manifest](https://headlamp.dev/docs/latest/installation/in-cluster/#using-simple-yaml) from GitHub and save it as a local file in your current working directory.

```shell
curl -s https://raw.githubusercontent.com/kubernetes-sigs/headlamp/main/kubernetes-headlamp.yaml > headlamp.yaml
```

Review the file and make the needed adjustments. Like namespace or service type. 
After making the necessary adjustments, apply the file to your cluster.
```shell
kubectl create namespace headlamp
kubectl apply -f ./headlamp.yaml
```

### Access Headlamp
#### Port-Forward
For quick access you can use port-forwarding:
```shell
kubectl port-forward -n headlamp service/headlamp 8080:80
```
Next access Headlamp at [http://127.0.0.1:8080](http://127.0.0.1:8080) in your browser.

#### Ingress
To setup the ingress server, you can use the [official Headlamp Ingress YAML](https://headlamp.dev/docs/latest/installation/in-cluster/#exposing-headlamp-with-an-ingress-server). The ingress file assumes that you have Contour and a cert-manager set up. If you don't, you'll not have TLS. 

Download the official Headlamp Ingress YAML from GitHub and save it as a local file in your current working directory. Review the file and make the needed adjustments. Like namespace, hostnames (`__URL__`), ingress class name, TLS settings and annotations. 
```shell
curl -s https://raw.githubusercontent.com/kubernetes-sigs/headlamp/main/kubernetes-headlamp-ingress-sample.yaml > headlamp-ingress.yaml
```

Apply the file on your cluster.
```shell
kubectl apply -f ./headlamp-ingress.yaml
```

### Authentication 
#### Helm
When you install using Helm, a default service account is created. The name of this service account matches the name from the Helm release name.

This service account has a cluster-admin cluster role binding by default (`headlamp-admin`). Consider implementing more restrict permissions: [Kubernetes RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

Check if the service account is created
```shell
kubectl get serviceaccount --namespace headlamp
```

Create a token directly for the service account:
```shell
kubectl create token headlamp-admin --namespace headlamp
```

Use this token to access Headlamp.

#### Yaml
If you used a YAML manifest installation, you still need to create the service account.
```shell
kubectl create serviceaccount headlamp-admin --namespace headlamp
```

Give permissions. Consider implementing more restrict permissions: [Kubernetes RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

```shell
kubectl create clusterrolebinding headlamp-admin --serviceaccount=headlamp:headlamp-admin --clusterrole=cluster-admin
```

Create a token directly for the service account:
```shell
kubectl create token headlamp-admin --namespace headlamp
```

Use this token to access Headlamp.
