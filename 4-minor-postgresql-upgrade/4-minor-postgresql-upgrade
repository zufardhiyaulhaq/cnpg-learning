# Minor Postgresql Upgrade

there are 2 type of postgresql version upgrade
1. supervised upgrade, where it will automatically upgrade all replicas, but will not continue with the current master. either switchover to replica or restart the current master will finish the upgrade process
2. unsupervised upgrade, where controller with automatically upgrade all replicas and master. there are 2 primary upgrade type (primaryUpdateMethod) here
   1. restart, if possible, perform an automated restart of the pod where the primary instance is running
   2. switchover, a switchover operation is automatically performed, setting the most aligned replica as the new target primary, and shutting down the former primary pod.

## Supervised Upgrade
1. create your postgresql cluster
```
kubectl apply -f cluster.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql-upgrade-test
  namespace: cnpg-system
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:17.2
  instances: 2
  storage:
    size: 1Gi
  primaryUpdateStrategy: supervised
```
2. upgrade the cluster
```
kubectl apply -f cluster.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql-upgrade-test
  namespace: cnpg-system
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:17.4
  instances: 2
  storage:
    size: 1Gi
  primaryUpdateStrategy: supervised
```
3. it will rollout restart replica but not primary, and waiting for us to switchover/delete the master pod.
```
echo-postgresql-upgrade-test-1             1/1     Running   0          4h22m
echo-postgresql-upgrade-test-2             1/1     Running   0          86s

zufar.dhiyaullhaq@MacBookPro 4-minor-postgresql-upgrade % k get cluster
NAME                           AGE     INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql-upgrade-test   4h32m   2           2       Waiting for user action    echo-postgresql-upgrade-test-1
```
4. switchover the master, there will be downtime for 5-10s
```
kubectl cnpg promote echo-postgresql-upgrade-test echo-postgresql-upgrade-test-2

zufar.dhiyaullhaq@MacBookPro 4-minor-postgresql-upgrade % k get cluster                                                                   
NAME                           AGE     INSTANCES   READY   STATUS                                       PRIMARY
echo-postgresql                8d      5           5       Cluster in healthy state                     echo-postgresql-8
echo-postgresql-upgrade-test   4h37m   2           1       Waiting for the instances to become active   echo-postgresql-upgrade-test-2
```
5. after replica became master. the old master (which is replica right now) will be automatically restarted
```
echo-postgresql-upgrade-test-1             1/1     Running   0          66s
echo-postgresql-upgrade-test-2             1/1     Running   0          7m28s
```

## Unsupervised Upgrade
I really recommed to use switchover method istead of restarting master since it's take more time for downtime

1. upgrade the cluster
```
kubectl apply -f cluster.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql-upgrade-test
  namespace: cnpg-system
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:17.5
  instances: 2
  storage:
    size: 1Gi
  primaryUpdateStrategy: unsupervised
  primaryUpdateMethod: switchover
```
2. operator will upgrade the replica, switchover the master to the newly upgraded replica, and upgrade the old master automatically. there will be downtime for 5-10s during the switchover.

https://cloudnative-pg.io/documentation/1.26/rolling_update/#automated-updates-unsupervised
https://cloudnative-pg.io/documentation/1.26/postgres_upgrades/

## Learning
1. unsupervised upgrade with restart primaryUpdateMethod take more than 1 minutes downtime of primary
2. switchover is prefered instead since it only take 5-10s downtime

## Issues

### Upgrade to unknown version
when upgrading to unknown version, replica pod will start crashing due to image not found/etc. to fix this, just update the version to the correct version and delete the replica pod which is in error state