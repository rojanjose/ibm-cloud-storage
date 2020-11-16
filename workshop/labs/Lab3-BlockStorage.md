
# Using IBM Cloud Block Storage with Kubernetes

The exercise demonstartes the use of Mongodb with IBM Cloud block storage.



## Mongodb with block storage

Create the namespace `mongodb` to for the helm chart to install the database.
```
kubectl create -f mongodb-namespace.yaml
namespace/mongodb created
```

### Helm Repo setup

Install Block storage plugin

```
helm repo add iks-charts https://icr.io/helm/iks-charts
helm repo update
helm install iks-charts/ibmcloud-block-storage-plugin
```

Block plugin pods.
```
kubectl get pod -n kube-system | grep block
ibmcloud-block-storage-driver-8j46j                   1/1     Running   0          10m
ibmcloud-block-storage-driver-bkscq                   1/1     Running   0          10m
ibmcloud-block-storage-plugin-54cd996d46-qhh75        1/1     Running   0          10m
```

Block storage class
```
kubectl get storageclasses | grep block
ibmc-block-bronze          ibm.io/ibmc-block   Delete          Immediate           true                   13m
ibmc-block-custom          ibm.io/ibmc-block   Delete          Immediate           true                   13m
ibmc-block-gold            ibm.io/ibmc-block   Delete          Immediate           true                   13m
ibmc-block-retain-bronze   ibm.io/ibmc-block   Retain          Immediate           true                   13m
ibmc-block-retain-custom   ibm.io/ibmc-block   Retain          Immediate           true                   13m
ibmc-block-retain-gold     ibm.io/ibmc-block   Retain          Immediate           true                   13m
ibmc-block-retain-silver   ibm.io/ibmc-block   Retain          Immediate           true                   13m
ibmc-block-silver          ibm.io/ibmc-block   Delete          Immediate           true                   13m
```

### Install manifests

Helm install dry run:
```
helm install ibmmongodb --set global.namespaceOverride=mongodb,auth.rootPassword=guest@dmin,auth.username=guestuser,auth.password=guestp@ss,auth.database=guestdb --debug --dry-run bitnami-ibm/mongodb > mongdb-install-dryrun.yaml
```

Review PVC and other parameters in the `mongdb-install-dryrun.yaml` file.

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

Pick values from [value.yaml](https://raw.githubusercontent.com/helm/charts/master/stable/mongodb/values.yaml)

Run the install in standalone mode.
```
helm install ibmmongodb --set global.namespaceOverride=mongodb,auth.rootPassword=guest@dmin,auth.username=guestuser,auth.password=guestp@ss,auth.database=guestdb bitnami-ibm/mongodb
```

Expected output:
```
❯ helm install ibmmongodb --set global.namespaceOverride=mongodb,auth.rootPassword=guest@dmin,auth.username=guestuser,auth.password=guestp@ss,auth.database=guestdb bitnami-ibm/mongodb

NAME: ibmmongodb
LAST DEPLOYED: Sat Nov 14 18:19:17 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

MongoDB can be accessed via port 27017 on the following DNS name(s) from within your cluster:

    ibmmongodb.mongodb.svc.cluster.local

To get the root password run:

    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongodb ibmmongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)

To get the password for "guestuser" run:

    export MONGODB_PASSWORD=$(kubectl get secret --namespace mongodb ibmmongodb -o jsonpath="{.data.mongodb-password}" | base64 --decode)

To connect to your database, create a MongoDB client container:

    kubectl run --namespace mongodb ibmmongodb-client --rm --tty -i --restart='Never' --image docker.io/bitnami/mongodb:4.4.1-debian-10-r61 --command -- bash

Then, run the following command:
    mongo admin --host "ibmmongodb" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace mongodb svc/ibmmongodb 27017:27017 &
    mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD

```

View the objects being created by the helm chart.

```
❯ kubectl get all -n mongodb
NAME                              READY   STATUS    RESTARTS   AGE
pod/ibmmongodb-7db76c6567-mkj2l   1/1     Running   0          2m48s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/ibmmongodb   ClusterIP   172.21.125.115   <none>        27017/TCP   2m49s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ibmmongodb   1/1     1            1           2m48s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/ibmmongodb-7db76c6567   1         1         1       2m48s
```

View the list of persistence volume claims. 
```
$ kubectl get pvc -n mongodb
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
ibmmongodb   Bound    pvc-1e82b3a7-ba42-46f5-a9df-21d8c7e28cf2   20Gi       RWO            ibmc-block-gold   91m
```

PV used by this PVC
```
kubectl get pv pvc-1e82b3a7-ba42-46f5-a9df-21d8c7e28cf2
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pvc-1e82b3a7-ba42-46f5-a9df-21d8c7e28cf2   20Gi       RWO            Delete           Bound    mongodb/ibmmongodb                           91m
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


Forward the port:
```
❯ kubectl port-forward --namespace mongodb svc/ibmmongodb 27017:27017 &
[1] 40848
Forwarding from 127.0.0.1:27017 -> 27017
Forwarding from [::1]:27017 -> 27017
```


Login using mongo client pod running on Kubernetes:

```
kubectl run --namespace mongodb ibmmongodb-client --rm --tty -i --restart='Never' --image docker.io/bitnami/mongodb:4.4.1-debian-10-r61 --command -- bash
If you don't see a command prompt, try pressing enter.
I have no name!@ibmmongodb-client:/$ mongo --version
MongoDB shell version v4.4.1
Build Info: {
    "version": "4.4.1",
    "gitVersion": "ad91a93a5a31e175f5cbf8c69561e788bbc55ce1",
    "openSSLVersion": "OpenSSL 1.1.1d  10 Sep 2019",
    "modules": [],
    "allocator": "tcmalloc",
    "environment": {
        "distmod": "debian10",
        "distarch": "x86_64",
        "target_arch": "x86_64"
    }
}
I have no name!@ibmmongodb-client:/$
I have no name!@ibmmongodb-client:/$ mongo admin --host "ibmmongodb" --authenticationDatabase admin -u root -p guest@dmin
MongoDB shell version v4.4.1
connecting to: mongodb://ibmmongodb:27017/admin?authSource=admin&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("c9aa6aab-354d-4bd7-a56f-84a401d689f7") }
MongoDB server version: 4.4.1
---
The server generated these startup warnings when booting:
        2020-11-15T03:11:02.936+00:00: ***** SERVER RESTARTED *****
        2020-11-15T03:11:02.943+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
---
---
        Enable MongoDB's free cloud-based monitoring service, which will then receive and display
        metrics about your deployment (disk utilization, CPU, operation statistics, etc).

        The monitoring data will be available on a MongoDB website with a unique URL accessible to you
        and anyone you share the URL with. MongoDB may use this information to make product
        improvements and to suggest MongoDB products and deployment options to you.

        To enable free monitoring, run the following command: db.enableFreeMonitoring()
        To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---
> db.guestbook.find()
>
> use guestbookdb
switched to db guestbookdb
> db.guestbook.find()
>
> db.createCollection("guestbook")
{ "ok" : 1 }
>
> db.guestbook.insertOne({"1":"Hello Kubernetes!"})
{
	"acknowledged" : true,
	"insertedId" : ObjectId("5fb09fa83d5e7bdd0f09f80b")
}
> db.guestbook.insertOne({"2":"Hola Kubernetes!"})
{
	"acknowledged" : true,
	"insertedId" : ObjectId("5fb09fb73d5e7bdd0f09f80c")
}
>
> db.guestbook.find()
{ "_id" : ObjectId("5fb09fa83d5e7bdd0f09f80b"), "1" : "Hello Kubernetes!" }
{ "_id" : ObjectId("5fb09fb73d5e7bdd0f09f80c"), "2" : "Hola Kubernetes!" }
>
>
> exit

```

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

