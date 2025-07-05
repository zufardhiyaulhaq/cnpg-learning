# WAL Storage

In most cases, having pg_wal on the same volume where PGDATA resides is fine. However, having WALs stored in a separate volume has a few benefits

1. Update Cluster with WAL storage
```
kubectl apply -f cluster.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql
  namespace: cnpg-system
spec:
  instances: 3
  storage:
    size: 20Gi
    storageClass: gtf-ack-essd-pl0-wait
  walStorage:
    size: 1Gi
    storageClass: gtf-ack-essd-pl0-wait
```
2. operator will start rollout restart the replica & master automatically. better to have `primaryUpdateStrategy: supervised` or `primaryUpdateStrategy: unsupervised` with `primaryUpdateMethod: switchover` before doing this activity to prevent master pod being terminated.
```
zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get pvc    
NAME                             STATUS    VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS            VOLUMEATTRIBUTESCLASS   AGE
echo-postgresql-13               Bound     d-k1a3qecdmzapcvkw4mol   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 21m
echo-postgresql-13-wal           Pending                                                      gtf-ack-essd-pl0-wait   <unset>                 2m29s
echo-postgresql-14               Bound     d-k1a6j280bbsxtsbcnyd6   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 17m
echo-postgresql-14-wal           Bound     d-k1a19hs0finsu6ev5e9d   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 2m29s
echo-postgresql-15               Bound     d-k1a4veew50n8oaidjvz2   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 14m
echo-postgresql-15-wal           Bound     d-k1a7u36tlbwd2cq8v7y7   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 2m29s

zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get cluster
NAME                           AGE     INSTANCES   READY   STATUS                                                     PRIMARY
echo-postgresql                8d      3           2       Primary instance is being restarted without a switchover   echo-postgresql-13

zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get pod
echo-postgresql-13                         0/1     Terminating   0          21m
echo-postgresql-14                         1/1     Running       0          2m10s
echo-postgresql-15                         1/1     Running       0          2m46s
```


https://cloudnative-pg.io/documentation/1.26/storage/#volume-for-wal