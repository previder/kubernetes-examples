# STaaS with NFS Subdir provisioner
### Overview
This example will install the [NFS subdir external provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) in the namespace `previder-staas`. This provisioner creates a subdir on the NFS for each claim.
### Install the NFS Subdir provisioner 
Replace your NFS endpoint and volume name in the required helm value fields.

```shell
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --namespace previder-staas \
    --create-namespace \
    --set nfs.server=<NFS endpoint> \
    --set nfs.path=<NFS path> \
    --set nfs.mountOptions[0]="nfsvers=3" \
    --set storageClass.name=previder-staas
```
The output of the command should look similar to this:
```text
LAST DEPLOYED: 
NAMESPACE: previder-staas
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
Check the NFS provisioner installation by running the following command:
```shell
kubectl -n previder-staas get pods
kubectl get storageclass
```
The pod should be running and there should be a storage class called `previder-staas`.
Using this storage class you can create a volume claim for a deployment.

### Volume claim
The storageClass `previder-staas` is now available for use in Persistent Volume Claims 
```shell
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: previder-staas
  resources:
    requests:
      storage: 5Gi
```

### NFS volume
When the volume claim has been created, you will see a created folder in the NFS volume. The default name format is: `${namespace}-${pvcName}-${pvName}`.  
This option and many more are configurable. See the [documentation of the provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/blob/master/charts/nfs-subdir-external-provisioner/values.yaml) for more information.
