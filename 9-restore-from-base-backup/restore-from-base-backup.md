# Restore from Base Backup


## Create Backup & Records for validation
1. create a record on database
2. check if record is exist
```
k port-forward svc/echo-postgresql 8080:8080

zufar.dhiyaullhaq@MacBookPro ~ % curl http://localhost:8080/postgresql/test
9e23a4cc-663b-4e3f-a7ef-29e590f57744:test
```
2. check if it's already applied
```
kubectl port-forward svc/echo-postgresql-rw -n cnpg-system 5432:5432

export PG_DB=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.dbname}' | base64 -d)
export PG_USER=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.user}' | base64 -d)
export PG_PASSWORD=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.password}' | base64 -d)

psql -h 127.0.0.1 -p 5432 -d app -U app

app=> SELECT * FROM echos WHERE echos.id = '9e23a4cc-663b-4e3f-a7ef-29e590f57744';
          created_at           |          updated_at           | deleted_at |                  id                  | echo
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-06 11:39:44.421576+00 | 2025-07-06 11:39:44.418815+00 |            | 9e23a4cc-663b-4e3f-a7ef-29e590f57744 | test
(1 row)
```
3. Execute on demand backup and wait until it's completed.
```
zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get backup                                                                                                                
NAME                                          AGE     CLUSTER           METHOD           PHASE       ERROR
echo-postgresql-on-demand-backup-02           18s     echo-postgresql   volumeSnapshot   completed  

zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get volumesnapshot 
NAME                                              READYTOUSE   SOURCEPVC                SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS                SNAPSHOTCONTENT  
echo-postgresql-on-demand-backup-02               true         echo-postgresql-14                               20Gi          alibabacloud-disk-snapshot   snapcontent-a1d3debc-5b7c-4034-8d01-5b9affa61b47   116s           117s
echo-postgresql-on-demand-backup-02-wal           true         echo-postgresql-14-wal                           1Gi           alibabacloud-disk-snapshot   snapcontent-c3087462-09c8-4ea0-aea5-fd1a7d0ee330   116s           117s
```
4. add another data after backup
```
curl http://localhost:8080/postgresql/testing-after-backup
c81b8c83-4b83-4a3a-947a-c57f55fe7f1f:testing-after-backup

app=> SELECT * FROM echos WHERE echos.id = 'c81b8c83-4b83-4a3a-947a-c57f55fe7f1f';
          created_at           |          updated_at           | deleted_at |                  id                  |         echo
-------------------------------+-------------------------------+------------+--------------------------------------+----------------------
 2025-07-06 11:44:43.839099+00 | 2025-07-06 11:44:43.836262+00 |            | c81b8c83-4b83-4a3a-947a-c57f55fe7f1f | testing-after-backup
(1 row)
```
## Create new Database from Volume Snapshot
1. create new database with volume snapshot
```
kubectl apply -f restore-with-volume-snapshot.yaml

  bootstrap:
    recovery:
      volumeSnapshots:
        storage:
          name: echo-postgresql-on-demand-backup-02
          kind: VolumeSnapshot
          apiGroup: snapshot.storage.k8s.io
        walStorage:
          name: echo-postgresql-on-demand-backup-02-wal
          kind: VolumeSnapshot
          apiGroup: snapshot.storage.k8s.io
```

2. Watch new cluster being created alongside it's replica
```
zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get cluster                                                                                                                           
NAME                                   AGE   INSTANCES   READY   STATUS                     PRIMARY
cluster-restore-with-volume-snapshot   4s    1                   Setting up primary 

zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get pod
NAME                                                             READY   STATUS      RESTARTS   AGE
cluster-restore-with-volume-snapshot-1                           0/1     Init:0/1    0          10s
cluster-restore-with-volume-snapshot-1-snapshot-recovery-nf7x6   0/1     Completed   0          34s

zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get pvc           
NAME                                         STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS            VOLUMEATTRIBUTESCLASS   AGE
cluster-restore-with-volume-snapshot-1       Bound    d-k1a3qecdmzapx7lcxhuq   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 17s
cluster-restore-with-volume-snapshot-1-wal   Bound    d-k1a3qecdmzapx7lcxhur   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 17s
```

3. Check records on the restored version, it will find record before backup but cannot find the record after backup is done.
```
kubectl port-forward svc/cluster-restore-with-volume-snapshot-rw -n cnpg-system 5432:5432

export PG_DB=$(kubectl get secret cluster-restore-with-volume-snapshot-app -o jsonpath='{.data.dbname}' | base64 -d)
export PG_USER=$(kubectl get secret cluster-restore-with-volume-snapshot-app -o jsonpath='{.data.user}' | base64 -d)
export PG_PASSWORD=$(kubectl get secret cluster-restore-with-volume-snapshot-app -o jsonpath='{.data.password}' | base64 -d)

psql -h 127.0.0.1 -p 5432 -d app -U app

app=> SELECT * FROM echos WHERE echos.id = '9e23a4cc-663b-4e3f-a7ef-29e590f57744';
          created_at           |          updated_at           | deleted_at |                  id                  | echo
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-06 11:39:44.421576+00 | 2025-07-06 11:39:44.418815+00 |            | 9e23a4cc-663b-4e3f-a7ef-29e590f57744 | test
(1 row)

app=> SELECT * FROM echos WHERE echos.id = 'c81b8c83-4b83-4a3a-947a-c57f55fe7f1f';
 created_at | updated_at | deleted_at | id | echo
------------+------------+------------+----+------
(0 rows)
```

https://cloudnative-pg.io/documentation/1.26/recovery