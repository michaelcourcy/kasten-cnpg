# Goal 

Create a kasten blueprint that trigger a barman backup on any cnpg cluster in your namespaces.

## Limitations 

Kasten and EDB has developped [a partnership to fully support EDB cluster](https://veeamkasten.dev/edb-and-kasten).
It includes : 
- fully automated backup and restore of EDB database
- Incremental backup based on snapshot 
- Support for fast restore based on local snapshot restore 
More technical details can be found [here](https://github.com/michaelcourcy/edb-kasten). 

But this is only available with EDB database not CNPG database.

With this blueprint you'll face those limitations :

1. The barmanObjectStore is defined idependently of a Kasten profile (no automatic integration)
2. The creation of a restored CNPG cluster is not supported by the blueprint and must be done independently (the blueprint only implement backup and delete action but not restore.)

# CloudNativePG

[CloudNativePG](https://cloudnative-pg.io/documentation/1.22/) CloudNativePG is an open source operator designed to manage PostgreSQL workloads on any supported Kubernetes cluster running in private, public, hybrid, or multi-cloud environments. CloudNativePG adheres to DevOps principles and concepts such as declarative configuration and immutable infrastructure.


## Installing the operator

The operator can be installed like any other resource in Kubernetes, through a YAML manifest applied via kubectl

```
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.24/releases/cnpg-1.24.0.yaml
```

You can verify the operator deployment is ok with:

```
kubectl get deployment -n cnpg-system cnpg-controller-manager
```

## Create a minio instance 

You can use any S3 object store but for our example we'll use a local ephemeral minio s3 server.
```
k create -f https://github.com/michaelcourcy/kasten-s3troubleshooting/raw/refs/heads/main/minio-dev.yaml
```

Access the minio UI 
```
kubectl --namespace kasten-io port-forward service/minio 9090:9090  
```
Open your browser on  http://localhost:9090. Connect with minio/minio123 and create a `barman` bucket.


## Deploy a PostgreSQL cluster

```
CLUSTER_NAME=cluster-example
SIZE=1Gi
SECRET=barman
NAMESPACE=my-cnpg-app
DESTINATION_PATH=s3://barman

kubectl create ns $NAMESPACE
kubectl config set-context --current --namespace=$NAMESPACE
kubectl create secret generic $SECRET --from-literal aws_access_key_id=minio --from-literal aws_secret_access_key=minio123

cat<<EOF |kubectl create -n $NAMESPACE -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: $CLUSTER_NAME
spec:
  instances: 3
  storage:
    size: $SIZE
  backup:
    barmanObjectStore:
      endpointURL: http://minio.kasten-io.svc.cluster.local:9000
      wal:
        compression: gzip
        # our minio install does not suport kms encryption         
        encryption: ""
      destinationPath: $DESTINATION_PATH
      s3Credentials:
        accessKeyId:
          name: $SECRET
          key: aws_access_key_id
        secretAccessKey:
          name: $SECRET
          key: aws_secret_access_key
EOF
```

Observe the change 
```
k get po -n $NAMESPACE  -w
```

If all go well you should see the status of the cluster heathy 
```
kubectl get cluster.postgresql.cnpg.io $CLUSTER_NAME
```

you should have this output
```
NAME              AGE     INSTANCES   READY   STATUS                     PRIMARY
cluster-example   7m35s   3           3       Cluster in healthy state   cluster-example-1
```

## creating some data 

```
kubectl exec -it $CLUSTER_NAME-1 -- bash
psql
```


We'll create a `links` table:
```
\c app

DROP TABLE IF EXISTS links;
CREATE TABLE links (
	id SERIAL PRIMARY KEY,
	url VARCHAR(255) NOT NULL,
	name VARCHAR(255) NOT NULL,
	description VARCHAR (255),
        last_update DATE
);
INSERT INTO links (url, name, description, last_update) VALUES('https://kasten.io','Kasten','Backup on kubernetes',NOW());
select * from links;
\q
exit
```

# Let's create a backup

Let's check that barman backup work as expected.

```
cat <<EOF| kubectl create -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
    namespace: $NAMESPACE
    name: backup-1
spec:
    method: barmanObjectStore
    cluster:
      name: $CLUSTER_NAME
EOF
```

If all go well you should see the status of your backup 
```
k get backup -n $NAMESPACE 
```

completed
```
NAME         AGE   CLUSTER             METHOD              PHASE       ERROR
backup-1     10s   cluster-example     barmanObjectStore   completed
```


# Test recovery 

Some data were corrupted and without deleting your actual cluster you need to revover your data 
as they were at the time of the backup.

For that we are creating another cluster `$CLUSTER_NAME-$BACKUP` reflecting those datas
```
BACKUP=$(kubectl get  -n $NAMESPACE backups.postgresql.cnpg.io -o jsonpath='{.items[0].metadata.name}')
cat<<EOF |kubectl create -n $NAMESPACE -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: $CLUSTER_NAME-$BACKUP
spec:
  instances: 3
  storage:
    size: $SIZE
  bootstrap:
    recovery:
      backup:
        name: $BACKUP  
EOF
```

You should see the lines you created in `$CLUSTER_NAME` initially

# Backup with kasten 

kasten can be extended with blueprint and blueprint-binding to perform the operations above on any cnpg database on 
your kubernetes cluster. 

- The blueprint will apply the operation above including deleting the backup when retention policy will remove restorepoint.
- The blueprint binding will bind any cnpg database to the blueprint

```
kubectl create -f cnpg-blueprint.yaml
kubectl create -f cnpg-blueprint-binding.yaml
```

# Test it 

With kasten create a backup of the namespace `my-cnpg-app` (define a profile for kanister action this is mandatory even if we don't define it) check if a new backup object has been created.

Delete the restorepoint and check if the backup object has been removed.

# Making sure you "backup" all the "backups"

The blueprint create a cnpg backup object during the blueprint backup action, but when blueprint backup action
are triggered Kasten has already captured all the specs and the new cnpg backup object won't be included in the 
restorepoint. 

You should create another policy that only capture cnpg backup object. This policy can have a high frequency because
it only capture cnpg backup objects.

# Restore after a disaster 

Let's simulate a complete disaster by deleting the namespace 
```
kubectl delete ns $NAMESPACE
```

Now all the backup objects are gone and cannot be restored because the operator will remove the status information at creation. 
We need to work from the restore point where we can extract status informations.

## Prepare with the restore point 

Go to the restorepoint and capture the spec of a backup using the UI. 
![capture spec](./capture-spec-in-rpc.png)

use jq to get a better output for instance 
```
backup='{"apiVersion":"postgresql.cnpg.io/v1","kind":"Backup","metadata":{"creationTimestamp":"2024-10-21T13:14:47Z","generation":1,"managedFields":[{"apiVersion":"postgresql.cnpg.io/v1","fieldsType":"FieldsV1","fieldsV1":{"f:spec":{".":{},"f:cluster":{".":{},"f:name":{}},"f:method":{}}},"manager":"kubectl-create","operation":"Update","time":"2024-10-21T13:14:47Z"},{"apiVersion":"postgresql.cnpg.io/v1","fieldsType":"FieldsV1","fieldsV1":{"f:status":{".":{},"f:backupId":{},"f:backupName":{},"f:beginLSN":{},"f:beginWal":{},"f:destinationPath":{},"f:endLSN":{},"f:endWal":{},"f:endpointURL":{},"f:instanceID":{".":{},"f:ContainerID":{},"f:podName":{}},"f:method":{},"f:phase":{},"f:s3Credentials":{".":{},"f:accessKeyId":{".":{},"f:key":{},"f:name":{}},"f:secretAccessKey":{".":{},"f:key":{},"f:name":{}}},"f:serverName":{},"f:startedAt":{},"f:stoppedAt":{}}},"manager":"manager","operation":"Update","subresource":"status","time":"2024-10-21T13:15:00Z"}],"name":"backup-1","namespace":"my-cnpg-app","resourceVersion":"5125201","uid":"db85b43e-f651-4069-b07c-e0381afae038"},"spec":{"cluster":{"name":"cluster-example"},"method":"barmanObjectStore"},"status":{"backupId":"20241021T131448","backupName":"backup-20241021131447","beginLSN":"0/5000028","beginWal":"000000010000000000000005","destinationPath":"s3://barman","endLSN":"0/6064A40","endWal":"000000010000000000000006","endpointURL":"http://minio.kasten-io.svc.cluster.local:9000","instanceID":{"ContainerID":"containerd://0b2dcdd36ee08f09e8f6de17c94dd82126d083f1d560c9dbf37e8fe887215624","podName":"cluster-example-2"},"method":"barmanObjectStore","phase":"completed","s3Credentials":{"accessKeyId":{"key":"aws_access_key_id","name":"barman"},"secretAccessKey":{"key":"aws_secret_access_key","name":"barman"}},"serverName":"cluster-example","startedAt":"2024-10-21T13:14:48Z","stoppedAt":"2024-10-21T13:14:59Z"}}'

echo $backup |jq '.status'
{
  "backupId": "20241021T131448",
  "backupName": "backup-20241021131447",
  "beginLSN": "0/5000028",
  "beginWal": "000000010000000000000005",
  "destinationPath": "s3://barman",
  "endLSN": "0/6064A40",
  "endWal": "000000010000000000000006",
  "endpointURL": "http://minio.kasten-io.svc.cluster.local:9000",
  "instanceID": {
    "ContainerID": "containerd://0b2dcdd36ee08f09e8f6de17c94dd82126d083f1d560c9dbf37e8fe887215624",
    "podName": "cluster-example-2"
  },
  "method": "barmanObjectStore",
  "phase": "completed",
  "s3Credentials": {
    "accessKeyId": {
      "key": "aws_access_key_id",
      "name": "barman"
    },
    "secretAccessKey": {
      "key": "aws_secret_access_key",
      "name": "barman"
    }
  },
  "serverName": "cluster-example",
  "startedAt": "2024-10-21T13:14:48Z",
  "stoppedAt": "2024-10-21T13:14:59Z"
}
```

This information will be useful later to restore the cluster. 

From the restore point now restore only the 2 secrets: 
1. barman: to access the minio cluster 
2. The app user to restart the database with the same credentials for the app database here it's `cluster-example-app`([this is not mandatory](https://cloudnative-pg.io/documentation/1.20/recovery/#additional-considerations))

![Restore only secrets](./restore-only-barman-and-app.png)

## Restore by recreating a cluster with a recovery bootstrap 

```
SIZE=1Gi
SECRET=barman
SUPER_SECRET=cluster-example-app
NAMESPACE=my-cnpg-app

SOURCE_CLUSTER=$(echo $backup |jq -r '.spec.cluster.name')
CLUSTER_NAME_RESTORED=$SOURCE_CLUSTER-restored
DESTINATION_PATH=$(echo $backup |jq -r '.status.destinationPath')
ENDPOINT_URL=$(echo $backup |jq -r '.status.endpointURL')
BACKUP_ID=$(echo $backup |jq -r '.status.backupId')

cat<<EOF |kubectl create -n $NAMESPACE -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: $CLUSTER_NAME_RESTORED
spec:
  superuserSecret: 
    name: $SUPER_SECRET
  instances: 3
  storage:
    size: $SIZE
  bootstrap:
    recovery:
      source: $SOURCE_CLUSTER
      recoveryTarget:
        backupID: $BACKUP_ID
        # if you need a PITR choose a date after the backup date
        # targetTime: "2024-10-21T13:14:48"        
  
  externalClusters:
    - name: $SOURCE_CLUSTER
      barmanObjectStore:
        endpointURL: $ENDPOINT_URL
        wal:
          compression: gzip
          # our minio install does not suport kms encryption         
          encryption: ""
        destinationPath: $DESTINATION_PATH
        s3Credentials:
          accessKeyId:
            name: $SECRET
            key: aws_access_key_id
          secretAccessKey:
            name: $SECRET
            key: aws_secret_access_key
EOF
```

You can now check in this cluster that you get back your links table with the one line you created previously.








