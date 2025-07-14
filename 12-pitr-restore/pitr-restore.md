# Point Time in Recovery Restoration

since we already have daily base backup with Kubernetes volume snapshot and WAL archival with object storage. we can restore Database with RPO (acceptable data loss) less than 5 minutes if we setup WAL archival less than 5 minutes.

for this Spike, we will configure archive_timeout to 10 minutes to better simulate the wal archival (while giving us time to create a new postgresql cluster from PITR & Base backup)

1. configure archive_timeout with 10m
```
spec:
  postgresql:
    parameters:
      archive_timeout: "10min"

kubectl apply -f cluster.yaml

app=> SHOW archive_timeout;
 archive_timeout
-----------------
 10min
(1 row)
```
2. check daily backup
```
k get backup 

NAME                                          AGE     CLUSTER           METHOD           PHASE       ERROR
echo-postgresql-daily-backup-20250714000000   15h     echo-postgresql   volumeSnapshot   completed   

k get volumesnapshot | grep echo-postgresql-daily-backup-20250714000000
NAME                                              READYTOUSE   SOURCEPVC                SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS                SNAPSHOTCONTENT                                    CREATIONTIME   AGE
echo-postgresql-daily-backup-20250714000000       true         echo-postgresql-13                               20Gi          alibabacloud-disk-snapshot   snapcontent-6c606a53-d3ff-4a91-b1d8-d4ed5353e1bd   15h            15h
echo-postgresql-daily-backup-20250714000000-wal   true         echo-postgresql-13-wal                           1Gi           alibabacloud-disk-snapshot   snapcontent-5a320ffa-a9ea-42ae-b8b9-a995b70dd1eb   15h            15h

```
3. make a record in database (this expect to be available on restore), and monitor if WAL is already archived
```
kubectl port-forward svc/echo-postgresql 8080:8080

curl http://localhost:8080/postgresql/test
15e0f592-ed82-42e2-86e2-471788f08b76:test

curl http://localhost:8080/postgresql/test
a8315bce-691d-4d73-a74e-452facc51b9d:test

curl http://localhost:8080/postgresql/test
aee3ed83-0227-4f15-aac0-301d9e1ee70b:test

k logs echo-postgresql-13 -c plugin-barman-cloud -f --tail 10
{"level":"info","ts":"2025-07-14T15:30:27.800156025Z","msg":"Archived WAL file","logging_pod":"echo-postgresql-13","walName":"/var/lib/postgresql/data/pgdata/pg_wal/00000007000000000000007C","startTime":"2025-07-14T15:30:26.419802835Z","endTime":"2025-07-14T15:30:27.800137304Z","elapsedWalTime":1.380334469}
```
4. once WAL is already archived, do another record on database (this expect to be miss on restore) and start the restore process by creating another postgresql cluster
```
curl http://localhost:8080/postgresql/test
5291e52a-7976-4bfd-8c2b-c24e17460633:test

curl http://localhost:8080/postgresql/test
21126dea-ebdc-4e32-be50-7a8ae67450ca:test

kubectl apply -f restore.yaml

k get cluster                
NAME                                   AGE   INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql-restore-pitr           5m20s   3           3       Cluster in healthy state   echo-postgresql-restore-pitr-1
```
5. new cluster will be created, exec into the postgresql and check the records, we can see records after WAL archival will not exist but any records before WAL archival process is exist on the cluster.
```
kubectl port-forward svc/echo-postgresql-restore-pitr-rw -n cnpg-system 5432:5432
export PG_PASSWORD=$(kubectl get secret echo-postgresql-restore-pitr-app -o jsonpath='{.data.password}' | base64 -d)

psql -h 127.0.0.1 -p 5432 -d app -U app

(old record, prove base backup is working)
app=> SELECT * FROM echos WHERE echos.id = '9e23a4cc-663b-4e3f-a7ef-29e590f57744';
          created_at           |          updated_at           | deleted_at |                  id                  | echo
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-06 11:39:44.421576+00 | 2025-07-06 11:39:44.418815+00 |            | 9e23a4cc-663b-4e3f-a7ef-29e590f57744 | test

(record before WAL archival, prove WAL archival & PITR is working)
app=> SELECT * FROM echos WHERE echos.id = 'aee3ed83-0227-4f15-aac0-301d9e1ee70b';
          created_at           |          updated_at           | deleted_at |                  id                  | echo
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-14 15:25:34.546384+00 | 2025-07-14 15:25:34.542902+00 |            | aee3ed83-0227-4f15-aac0-301d9e1ee70b | test

(record created after WAL archival, this is data that is loss)
app=> SELECT * FROM echos WHERE echos.id = '5291e52a-7976-4bfd-8c2b-c24e17460633';
 created_at | updated_at | deleted_at | id | echo
------------+------------+------------+----+------
(0 rows)
```

https://cloudnative-pg.io/documentation/1.26/recovery/
https://cloudnative-pg.io/documentation/1.26/wal_archiving/
https://cloudnative-pg.io/documentation/1.26/backup/

