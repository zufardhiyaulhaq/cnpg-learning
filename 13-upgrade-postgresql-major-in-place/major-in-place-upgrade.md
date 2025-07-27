# Upgrade Postgresql In Place Major

1. create deployment & cluster for postgresql major upgrade, we will use postgresql 16.9
```
kubectl apply -f cluster.yaml
kubectl apply -f deployment.yaml
```
2. test the flow
```
 k get cluster
NAME                            AGE   INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql-major-upgrade   30m   2           2       Cluster in healthy state   echo-postgresql-major-upgrade-1

k port-forward svc/echo-postgresql-upgrade 8080:8080

curl http://localhost:8080/postgresql/test
cc1f267b-6a12-4a48-8d24-3759c439fbc3:test

kubectl cnpg psql echo-postgresql-major-upgrade

\c app
SELECT * FROM echos WHERE echos.id = 'cc1f267b-6a12-4a48-8d24-3759c439fbc3';

app=# SELECT * FROM echos WHERE echos.id = 'cc1f267b-6a12-4a48-8d24-3759c439fbc3';
          created_at           |          updated_at           | deleted_at |                  id                  | echo 
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-27 09:00:16.161352+00 | 2025-07-27 09:00:16.156363+00 |            | cc1f267b-6a12-4a48-8d24-3759c439fbc3 | test
```

3. Start upgrading the postgresql from 16.9 to 17.5
```
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql-major-upgrade
  namespace: cnpg-system
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:17.5

kubectl apply -f cluster.yaml
```
all pod will be terminated, but PVC and PV will intact
```
echo-postgresql-major-upgrade-1       Bound    d-k1a01de7hxbnngy4aasr   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 46m
echo-postgresql-major-upgrade-1-wal   Bound    d-k1ai3fx1779v052evzd6   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 46m
echo-postgresql-major-upgrade-2       Bound    d-k1a2eruez5v4uwzn2yhv   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 45m
echo-postgresql-major-upgrade-2-wal   Bound    d-k1a6lqr238ym37hsl68o   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 45m
```
4. Monitoring the upgrade
   
cluster status will change to "Upgrading Postgres major version"
```
k get cluster

echo-postgresql-major-upgrade   48m   2                   Upgrading Postgres major version   echo-postgresql-major-upgrade-1
```
cloudnative-pg will start by enabling the master pod. after master pod is started successfully, we can start write/read to the master

5. Check the upgrade
```
kubectl get cluster 
NAME                            AGE   INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql-major-upgrade   53m   2           2       Cluster in healthy state   echo-postgresql-major-upgrade-1

kubectl cnpg status echo-postgresql-major-upgrade
Cluster Summary
Name                 cnpg-system/echo-postgresql-major-upgrade
System ID:           7531688940398657563
PostgreSQL Image:    ghcr.io/cloudnative-pg/postgresql:17.5
Primary instance:    echo-postgresql-major-upgrade-1
```

6. Test flow and validate existing and new data exist
```
k port-forward svc/echo-postgresql-upgrade 8080:8080

curl http://localhost:8080/postgresql/test
19f720e1-d12c-4230-840b-8700a606be97:test

 kubectl cnpg psql echo-postgresql-major-upgrade
psql (17.5 (Debian 17.5-1.pgdg110+1))
Type "help" for help.

postgres=# \c app
You are now connected to database "app" as user "postgres".
app=# SELECT * FROM echos WHERE echos.id = 'cc1f267b-6a12-4a48-8d24-3759c439fbc3';
          created_at           |          updated_at           | deleted_at |                  id                  | echo 
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-27 09:00:16.161352+00 | 2025-07-27 09:00:16.156363+00 |            | cc1f267b-6a12-4a48-8d24-3759c439fbc3 | test
(1 row)

app=# 
app=# SELECT * FROM echos WHERE echos.id = '19f720e1-d12c-4230-840b-8700a606be97';
          created_at           |          updated_at           | deleted_at |                  id                  | echo 
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-27 09:22:19.762326+00 | 2025-07-27 09:22:19.755823+00 |            | 19f720e1-d12c-4230-840b-8700a606be97 | test
(1 row)
```

