# Base Backup

There are 2 main component to archive disaster recovery in Postgresql.
1. Physical base backups: a copy of all the files that PostgreSQL uses to store the data in the database
2. WAL archive: a location containing the WAL files (transactional logs) that are continuously written by Postgres and archived for data durability

PostgreSQL instances can be restarted from a base backup without the need of a WAL archive, even though they can take advantage of it, if available to archive continuous backup and ensure RPO is less than 5 minutes. WAL archive alone is useless. Without a physical base backup, you cannot restore a PostgreSQL cluster.

There are 2 storage location of base backup
1. Kubernetes Volume Snapshots
2. Object Store (via barman plugin)

## Create backup with Volume Snapshots
1. make sure kubernetes support volume snapshot
```
kubectl get volumesnapshotclass
```
2. update cluster manifest with backup configuration
```
  backup:
    target: prefer-standby
    volumeSnapshot:
      className: alibabacloud-disk-snapshot
      online: false
```

notes:
- target `prefer-standby` will run backup on most updated replica.
- online `false` mean that replica will bring to shutdown before taking backup

3. Create on demand base backup
```
kubectl apply -f on-demand-backup.yaml
```
4. replica will be fencing
```
echo-postgresql-14                         0/1     Running   0          9h

Events:
  Type    Reason    Age                From                   Message
  ----    ------    ----               ----                   -------
  Normal  FencePod  27s (x2 over 27s)  cloudnative-pg-backup  Fencing Pod echo-postgresql-14
```
5. after backup completed, replica pod started to run back, and volumesnapshot is created.
```
zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get backup     
NAME                                  AGE   CLUSTER           METHOD           PHASE       ERROR
echo-postgresql-on-demand-backup-01   71s   echo-postgresql   volumeSnapshot   completed  

zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get volumesnapshot
NAME                                      READYTOUSE   SOURCEPVC                SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS                SNAPSHOTCONTENT                                    CREATIONTIME   AGE
echo-postgresql-on-demand-backup-01       true         echo-postgresql-14                               20Gi          alibabacloud-disk-snapshot   snapcontent-23d0192c-1ad6-4f7f-a980-6ef10b405d9b   59s            59s
echo-postgresql-on-demand-backup-01-wal   true         echo-postgresql-14-wal                           1Gi           alibabacloud-disk-snapshot   snapcontent-a1aa45d8-845d-48d5-8080-3b7899768b58   59s            59s
```

## Create Scheduled Backup with Volume Snapshot
1. after configuring the backup configuration on cluster object. you can create the scheduled backup
```
kubectl apply -f scheduled-backup.yaml

apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: echo-postgresql-daily-backup
  namespace: cnpg-system
spec:
  immediate: true
  schedule: "0 0 0 * * *"
  method: volumeSnapshot
  cluster:
    name: echo-postgresql
```
this will run backup every night and run the first execution immediately.
2. scheduled backup and backup will created, similar process of fencing the replica will happen
```
zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get scheduledbackup
NAME                           AGE   CLUSTER           LAST BACKUP
echo-postgresql-daily-backup   66s   echo-postgresql   66s

zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get backup
NAME                                          AGE   CLUSTER           METHOD           PHASE       ERROR
echo-postgresql-daily-backup-20250706040410   59s   echo-postgresql   volumeSnapshot   completed   
echo-postgresql-on-demand-backup-01           9h    echo-postgresql   volumeSnapshot   completed

zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get volumesnapshot                                           
NAME                                              READYTOUSE   SOURCEPVC                SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS                SNAPSHOTCONTENT                                    CREATIONTIME   AGE
echo-postgresql-daily-backup-20250706040410       true         echo-postgresql-14                               20Gi          alibabacloud-disk-snapshot   snapcontent-d343baa7-9157-4b9b-bd73-3b86383531d0   89s            89s
echo-postgresql-daily-backup-20250706040410-wal   true         echo-postgresql-14-wal                           1Gi           alibabacloud-disk-snapshot   snapcontent-82fe6532-438c-4d7d-bc54-9c93b7ed72a5   89s            89s
```

https://cloudnative-pg.io/documentation/1.26/backup/
https://cloudnative-pg.io/documentation/1.26/cloudnative-pg.v1/#postgresql-cnpg-io-v1-VolumeSnapshotConfiguration
https://cloudnative-pg.io/documentation/1.26/recovery/#how-recovery-works-under-the-hood
https://cloudnative-pg.io/documentation/1.26/appendixes/backup_volumesnapshot/#persistence-of-volume-snapshot-objects