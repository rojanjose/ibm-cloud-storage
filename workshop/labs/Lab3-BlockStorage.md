
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

## Mongodb with block storage

### Install manifests

Pick values from [value.yaml](https://raw.githubusercontent.com/helm/charts/master/stable/mongodb/values.yaml)
Installed in `standalone` mode.
Dryrun: 

TODO: fix security content values for OpenShift.
```
helm install mongo bitnami/mongodb --set podSecurityContext.fsGroup=1001020000,containerSecurityContext.runAsUser=1001020000 --debug --dry-run > mongdb-install-dryrun.yaml

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

### Install Mongodb

TODO: fix security content values for OpenShift.
```
helm install mongo bitnami/mongodb --set podSecurityContext.fsGroup=1001020000,containerSecurityContext.runAsUser=1001020000 --debug
```

Expected output:
```
$ helm install mongo bitnami/mongodb --set podSecurityContext.fsGroup=1001020000,containerSecurityContext.runAsUser=1001020000 --debug

install.go:159: [debug] Original chart version: ""
install.go:176: [debug] CHART PATH: /Users/rojan/Library/Caches/helm/repository/mongodb-9.2.4.tgz

client.go:108: [debug] creating 5 resource(s)
NAME: mongo
LAST DEPLOYED: Thu Oct 15 12:49:06 2020
NAMESPACE: mongo-user001
STATUS: deployed
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
containerSecurityContext:
  runAsUser: 1001020000
podSecurityContext:
  fsGroup: 1001020000

......

```

View the objects being created by the helm chart.

```
$ oc get all 
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/mongo-mongodb   ClusterIP   172.21.242.70   <none>        27017/TCP   17s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mongo-mongodb   0/1     0            0           17s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/mongo-mongodb-6f8f7cd789   1         0         0       17s
```

View the list of persistence volume claims. Note that the `mongo-mongodb` is pending volume allocation.

```
$ oc get pvc    

NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mongo-mongodb   Pending                                      ibmc-block-gold   21s
```

After waiting for some time. The pod supporting Mongodb is in `Running` status.

```
$ oc get all       
NAME                                 READY   STATUS    RESTARTS   AGE
pod/mongo-mongodb-66d7bcd7cf-vqvbj   1/1     Running   0          8m37s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/mongo-mongodb   ClusterIP   172.21.242.70   <none>        27017/TCP   12m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mongo-mongodb   1/1     1            1           12m

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/mongo-mongodb-66d7bcd7cf   1         1         1       8m37s
replicaset.apps/mongo-mongodb-6f8f7cd789   0         0         0       12m
```

And the PVC `mongo-mongodb` is now bound to volume `pvc-2f423668-4f87-4ae4-8edf-8c892188b645`
```
$ oc get pvc 
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mongo-mongodb   Bound    pvc-2f423668-4f87-4ae4-8edf-8c892188b645   20Gi       RWO            ibmc-block-gold   2m26s
```


## Accessing data

### From application

NOTES:
** Please be patient while the chart is being deployed **

MongoDB can be accessed via port 27017 on the following DNS name(s) from within your cluster:

    mongo-mongodb.mongo-user001.svc.cluster.local

To get the root password run:

    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongo-user001 mongo-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)

To connect to your database, create a MongoDB client container:

    kubectl run --namespace mongo-user001 mongo-mongodb-client --rm --tty -i --restart='Never' --image docker.io/bitnami/mongodb:4.4.1-debian-10-r13 --command -- bash

Then, run the following command:
    mongo admin --host "mongo-mongodb" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace mongo-user001 svc/mongo-mongodb 27017:27017 &
    mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD


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

