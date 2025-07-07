# CloudNativePG Learning

## Learning
1. always implement primaryUpdateStrategy & primaryUpdateMethod. this to avoid master pod being deleted for any activity (upgrade, adding wal storage, etc). switchover is prefered since it's faster instead of waiting Kubernetes master pod to shutdown and start back.
```
spec:
  primaryUpdateStrategy: unsupervised
  primaryUpdateMethod: switchover
```
2. split WAL storage to speed up I/O performance and you can fine tune the storageClass & performance for PGDATA and WAL.
```
  storage:
    size: 20Gi
    storageClass: gtf-ack-essd-pl0-wait
  walStorage:
    size: 1Gi
    storageClass: gtf-ack-essd-pl0-wait
```
3. always check the image is exist to prevent any distruption during activity (issue during activity is more hard to fix). https://github.com/cloudnative-pg/postgres-containers/pkgs/container/postgresql
4. always plan which storageclass to choose and make sure it's support online volume resizing for easier increase of disk size
5. always plan to which size during disk increase since reducing disk size is not possible.
6. Activity that required master switchover/restart
   1. adding WAL Storage
   2. Postgresql Upgrade
   3. Change StorageClass
   4. Change resource request & limit spcification
   5. TBD
7. Activity that not required master switchover/restart
   1. modify synchronous replica
   2. increase disk
8. Replication slot are enable by default https://cloudnative-pg.io/documentation/1.26/replication/#replication-slots

## Cheatsheet

### Switchover master
```
kubectl get pod
kubectl get clusters.postgresql.cnpg.io

kubectl cnpg promote <cluster-name> <replica-pod-to-be-promoted>
```

### Check Synchronous Replica
```
app=> SHOW synchronous_commit;
app=> SHOW synchronous_standby_names;
```