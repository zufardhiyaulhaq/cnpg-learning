# Minor Postgresql Downgrade

similar like minor postgresql downgrade, it's also respect the primaryUpdateStrategy & primaryUpdateStrategy.

1. downgrade the cluster
```
kubectl apply -f cluster.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql-upgrade-test
  namespace: cnpg-system
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:17.0
  instances: 2
  storage:
    size: 1Gi
  primaryUpdateStrategy: supervised
```
2. since this is supervised. replica will be automatically downgraded, and wait manual intervention to delete master pod or switchover
```
zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get cluster
NAME                           AGE     INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql-upgrade-test   4h53m   2           2       Waiting for user action    echo-postgresql-upgrade-test-1

echo-postgresql-upgrade-test-1             1/1     Running   0          12m
echo-postgresql-upgrade-test-2             1/1     Running   0          4m8s

zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get pod echo-postgresql-upgrade-test-2 -oyaml | grep image
    image: ghcr.io/cloudnative-pg/postgresql:17.0

zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get pod echo-postgresql-upgrade-test-1 -oyaml | grep image
    image: ghcr.io/cloudnative-pg/postgresql:17.5
```
3. switchover the master, there will be downtime for 5-10s
```
kubectl cnpg promote echo-postgresql-upgrade-test echo-postgresql-upgrade-test-2

zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get cluster                                                                   
NAME                           AGE     INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql                8d      5           5       Cluster in healthy state   echo-postgresql-8
echo-postgresql-upgrade-test   4h55m   2           1       Switchover in progress     echo-postgresql-upgrade-test-2

zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get pod echo-postgresql-upgrade-test-2 -oyaml | grep image
    image: ghcr.io/cloudnative-pg/postgresql:17.0

zufar.dhiyaullhaq@MacBookPro 5-minor-postgresql-downgrade % k get pod echo-postgresql-upgrade-test-1 -oyaml | grep image
    image: ghcr.io/cloudnative-pg/postgresql:17.0
```