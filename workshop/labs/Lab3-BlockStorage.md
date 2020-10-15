
# Using IBM Cloud Block Storage with Kubernetes


Log into the OpenShift cluster and create a project where we want to deploy Mongodb. Repleace `<username>` with the assigned lab user.

```
oc new-project mongo-<usename> 
```
Expected output:
```
$ oc new-project mongo-user001

Now using project "mongo-user001" on server "https://c100-e.us-south.containers.cloud.ibm.com:30817".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app ruby~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```

## Helm Repo setup

The lab uses Bitnami's Mongodb [Helm chart](https://github.com/helm/charts/tree/master/stable/mongodb) to show case the use of block storage. Set the Bitnami helm repo prior to installing mongodb.

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Expected output:
```
$ helm repo add bitnami https://charts.bitnami.com/bitnami

"bitnami" has been added to your repositories
```
Validate the repo is available in the list.
```
$ helm repo list

NAME      	URL
bitnami   	https://charts.bitnami.com/bitnami
iks-charts	https://icr.io/helm/iks-charts
```

## Install Mongodb

Pick values from [value.yaml](https://raw.githubusercontent.com/helm/charts/master/stable/mongodb/values.yaml)
Installed in `standalone` mode.
Dryrun: 
```
helm install mongo bitnami/mongodb --debug --dry-run > mongdb-install-dryrun.yaml
```

View the output of the command. Review the section that manifests PVC provisioing. 
Spec list only parameters, Access mode `RWO` and storage size `8gi`.
Defualt values are assumed for remaining parmeters such as storage class, ... 


```
# Source: mongodb/templates/standalone/pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mongo-mongodb
  namespace: mongo-user001
  labels:
    app.kubernetes.io/name: mongodb
    helm.sh/chart: mongodb-9.2.4
    app.kubernetes.io/instance: mongo
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: mongodb
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "8Gi"
```

## Accessing data

### From application

### For cluster


## Cleanup

This part of the lab desrcibes the steps to delete block storage. Prior to deleting the storage, the pods mounting the volumes should be removed. Start with uninstalling Mongodb and deleting the volumes.

### Uninstall mongodb
## Delete Block Storage

Get PVCs:

```
$ oc get pvc

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
datadir-mongo-mongodb-secondary-0   Bound    pvc-84d86f6b-7330-4f6c-9561-af96bce35f39   20Gi       RWO            ibmc-block-gold   4h38m
mongo-mongodb                       Bound    pvc-21ff3768-e198-43e7-8bb3-4344c3a68aa9   20Gi       RWO            ibmc-block-gold   4h8m
```

Get PVs:

```
oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                                                                     STORAGECLASS       REASON   AGE
pvc-21ff3768-e198-43e7-8bb3-4344c3a68aa9   20Gi       RWO            Delete           Bound    mongodb-helm/mongo-mongodb                                                                                                                            4h8m
pvc-84d86f6b-7330-4f6c-9561-af96bce35f39   20Gi       RWO            Delete           Bound    mongodb-helm/datadir-mongo-mongodb-secondary-0                                                                                                        4h38m
pvc-ff865949-740b-45b9-a498-0af27350d079   24Gi       RWX            Delete           Bound    default/silver-pvc                                                                                                        ibmc-file-silver            21h
```

Delete PVC

```
$ oc delete pvc datadir-mongo-mongodb-secondary-0 

persistentvolumeclaim "datadir-mongo-mongodb-secondary-0" deleted
```

Check PVC:
```
$ oc get pvc

NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mongo-mongodb   Bound    pvc-21ff3768-e198-43e7-8bb3-4344c3a68aa9   20Gi       RWO            ibmc-block-gold   4h15m
```

```
$ oc get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                                                                     STORAGECLASS       REASON   AGE
pvc-21ff3768-e198-43e7-8bb3-4344c3a68aa9   20Gi       RWO            Delete           Bound    mongodb-helm/mongo-mongodb                                                                                                                            4h14m
pvc-ff865949-740b-45b9-a498-0af27350d079   24Gi       RWX            Delete           Bound    default/silver-pvc                                                                                                        ibmc-file-silver            22h
```

Check the volume:
```
$ ibmcloud sl block volume-list --columns id --columns notes | grep pvc-84d86f6b-7330-4f6c-9561-af96bce35f39

177939378   {"plugin":"ibmcloud-block-storage-plugin-68d5c65db9-mxbxd","region":"us-south","cluster":"bthmtf3d0tp8ib025g50","type":"Endurance","ns":"mongodb-helm","pvc":"datadir-mongo-mongodb-secondary-0","pv":"pvc-84d86f6b-7330-4f6c-9561-af96bce35f39","storageclass":"ibmc-block-gold","reclaim":"Delete"}
```

Cancel volume:
```
$ ibmcloud sl block volume-cancel 177939378

This will cancel the block volume: 177939378 and cannot be undone. Continue?> yes
Failed to cancel block volume: 177939378.
No billing item is found to cancel.
```

