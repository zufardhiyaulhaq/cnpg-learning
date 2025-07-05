# Simple Cluster

1. creating postgresql cluster
```
kubectl apply -f cluster.yaml
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
7. create deployment
```
kubectl apply -f deployment.yaml
kubectl get pod
```
8. port forward service & test curl
```
kubectl port-forward svc/echo-postgresql 8080:8080
while true; do curl http://localhost:8080/postgresql/test; sleep 0.5; done
```
9. port forward database & check table
```
kubectl port-forward svc/echo-postgresql-rw -n cnpg-system 5432:5432

export PG_DB=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.dbname}' | base64 -d)
export PG_USER=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.user}' | base64 -d)
export PG_PASSWORD=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.password}' | base64 -d)

psql -h 127.0.0.1 -p 5432 -d app -U app

zufar.dhiyaullhaq@MacBookPro 1-simple-cluster % psql -h 127.0.0.1 -p 5432 -d app -U app
Password for user app: 
psql (14.18 (Homebrew), server 17.5 (Debian 17.5-1.pgdg110+1))
WARNING: psql major version 14, server major version 17.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

app=> \dt
       List of relations
 Schema | Name  | Type  | Owner 
--------+-------+-------+-------
 public | echos | table | app
(1 row)

app=> select * from echos;
          created_at           |          updated_at           | deleted_at |                  id                  | echo 
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-07-04 15:01:50.560357+00 | 2025-07-04 15:01:50.554699+00 |            | b538474e-1106-42f3-a8c6-345449606158 | test
 2025-07-04 15:01:51.401685+00 | 2025-07-04 15:01:51.399392+00 |            | c72a1b22-4660-4cd8-9c75-808335729df5 | test
```


https://cloudnative-pg.io/documentation/1.26/quickstart/
https://cloudnative-pg.io/documentation/1.26/applications/