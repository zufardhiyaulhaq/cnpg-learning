# WAL Archival

when implementing WAL Archival, there is only 1 data source that we can use which is object storage. this is implemented via barman cloud plugin https://cloudnative-pg.io/plugin-barman-cloud/docs/installation/.

1. Install Barman Cloud Plugin
2. Install minio (if required) & create bucket by accessing the minio portal
```
kubectl apply -f minio.yaml

kubectl port-forward svc/minio 9001:9001
http://127.0.0.1:9001/
```
3. Create object store & required secret
```
kubectl apply -f secret-example.yaml
kubectl apply -f object-store-example.yaml
```
1. apply the cluster with updated plugin
```
kubectl apply -f cluster.yaml

spec:
  plugins:
  - name: barman-cloud.cloudnative-pg.io
    isWALArchiver: true
    parameters:
      barmanObjectName: s3-object-store-echo-postgresql
```
4. Cluster will start the process of installing the plugin with sidecar, required master and replica restart
```
NAME                                   AGE    INSTANCES   READY   STATUS                                       PRIMARY
echo-postgresql                        10d    3           2       Waiting for the instances to become active   echo-postgresql-14

$ kubectl get pod
echo-postgresql-13                              1/1     Running   0          87m
echo-postgresql-14                              1/1     Running   0          88m
echo-postgresql-15                              1/2     Running   0          111s
```
5. We will see the primary postgresql logs (container plugin-barman-cloud) will start sending WAL archive to S3/Minio, while replica is not doing any WAL archival
```
k logs echo-postgresql-15 -c plugin-barman-cloud -f --tail 10

{"level":"info","ts":"2025-07-13T05:10:42.805540985Z","msg":"Applying backup retention policy","logging_pod":"echo-postgresql-15","retentionPolicy":"7d"}
{"level":"info","ts":"2025-07-13T05:11:08.709343548Z","msg":"Executing barman-cloud-wal-archive","logging_pod":"echo-postgresql-15","walName":"/var/lib/postgresql/data/pgdata/pg_wal/000000060000000000000073","options":["--gzip","--endpoint-url","http://minio:9000/","--cloud-provider","aws-s3","s3://wal-archival/","echo-postgresql","/var/lib/postgresql/data/pgdata/pg_wal/000000060000000000000073"]}
{"level":"info","ts":"2025-07-13T05:11:10.263064945Z","msg":"Archived WAL file","logging_pod":"echo-postgresql-15","walName":"/var/lib/postgresql/data/pgdata/pg_wal/000000060000000000000073","startTime":"2025-07-13T05:11:08.709335058Z","endTime":"2025-07-13T05:11:10.263039494Z","elapsedWalTime":1.553704436}

k logs echo-postgresql-14 -c plugin-barman-cloud -f --tail 10

{"level":"info","ts":"2025-07-13T05:15:44.062401857Z","msg":"Skipping retention policy enforcement, not the current primary","logging_pod":"echo-postgresql-14","currentPrimary":"echo-postgresql-15","podName":"echo-postgresql-14"}
```