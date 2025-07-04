# Simple Cluster

1. creating postgresql cluster
```
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql
  namespace: cnpg-system
spec:
  instances: 4
  storage:
    size: 1Gi
```
2. postgresql get cluster will show the master IP
```
zufar.dhiyaullhaq@Zufar-Dhiyaulhaq github % k get cluster
NAME              AGE    INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql   7d6h   4           4       Cluster in healthy state   echo-postgresql-1
```
3. pod is created
```
zufar.dhiyaullhaq@Zufar-Dhiyaulhaq github % k get pod
NAME                                   READY   STATUS    RESTARTS   AGE
cnpg-cloudnative-pg-7bf9fd4554-52tqk   1/1     Running   0          3d17h
echo-postgresql-1                      1/1     Running   0          7d6h
echo-postgresql-2                      1/1     Running   0          3d6h
echo-postgresql-3                      1/1     Running   0          7d5h
echo-postgresql-4                      1/1     Running   0          7d5h
```
4. service is created
```
zufar.dhiyaullhaq@Zufar-Dhiyaulhaq github % k get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
cnpg-webhook-service   ClusterIP   172.16.2.65     <none>        443/TCP    7d6h
echo-postgresql-r      ClusterIP   172.16.10.189   <none>        5432/TCP   7d6h
echo-postgresql-ro     ClusterIP   172.16.6.253    <none>        5432/TCP   7d6h
echo-postgresql-rw     ClusterIP   172.16.7.245    <none>        5432/TCP   7d6h
```
5. there is secret created for this postgresql
```
zufar.dhiyaullhaq@Zufar-Dhiyaulhaq github % k get secret echo-postgresql-app -oyaml
apiVersion: v1
data:
  dbname: YXBw
  host: ZWNoby1wb3N0Z3Jlc3FsLXJ3
  jdbc-uri: xxx
  password: yyy
  pgpass: zzz
  port: aaa
  uri: bbb
  user: YXBw
  username: YXBw
```
6. by default, it's create database "app" and user "app" if it's not defined.


https://cloudnative-pg.io/documentation/1.26/quickstart/
https://cloudnative-pg.io/documentation/1.26/applications/