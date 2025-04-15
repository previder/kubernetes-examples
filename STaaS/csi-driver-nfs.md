# STaaS with CSI driver NFS
### Overview
This example will install the [CSI driver NFS](https://github.com/kubernetes-csi/csi-driver-nfs) in the namespace `previder-staas`. This provisioner has one NFS mount and can use a NFS type volume or Persistent Volume Claims. This example will only use the PVC setup as the NFS type volume limits your namespace Pod Security Standard.  

**Notice:** Using this CSI driver, you will get access to the NFS volume root and using the Deployment settings determine a path inside. This can be a security issue.  

### Install the CSI driver NFS
Follow the [installation guide](https://github.com/kubernetes-csi/csi-driver-nfs/tree/master/charts) of the CSI driver NFS, or execute the commands below.

```shell
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
    --namespace previder-staas \
    --create-namespace \
    --version v4.5.0
```

The output of the command should look similar to this:
```text
NAME: csi-driver-nfs
LAST DEPLOYED: 
NAMESPACE: previder-staas
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The CSI NFS Driver is getting deployed to your cluster.

To check CSI NFS Driver pods status, please run:

  kubectl --namespace=previder-staas get pods --selector="app.kubernetes.io/instance=csi-driver-nfs" --watch
```

Create a storageClass
```shell
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: previder-staas-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: <NFS endpoint>
  share: <NFS share>
reclaimPolicy: Delete
volumeBindingMode: Immediate
```
### Volume Claim
Create a Persistent Volume Claim

```shell
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: previder-staas-csi
```
