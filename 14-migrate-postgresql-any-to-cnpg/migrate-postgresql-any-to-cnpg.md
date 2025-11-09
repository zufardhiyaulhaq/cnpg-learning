# Migrate any Postgresql to CNPG
We will use logical replication to migrate any postgresql to CNPG cluster.

### Preparation
we need to prepare postgresql outside of CNPG that will be used as subscriber.
1. create the cluster
```
kubectl apply -f posgresql.yaml
```
2. check if wal_level is already log
```
k exec -it postgresql-publisher-786dfc64f7-k6zdv -- bash

root@postgresql-publisher-786dfc64f7-k6zdv:/# psql -U echo_user -d echo
psql (15.14 (Debian 15.14-1.pgdg13+1))
Type "help" for help.

echo=# show wal_level;
 wal_level
-----------
 logical
(1 row)
```
3. create replication user and their permission
```
CREATE USER logical_repl WITH REPLICATION LOGIN PASSWORD 'repl_password';

GRANT CONNECT ON DATABASE echo TO logical_repl;
GRANT USAGE ON SCHEMA public TO logical_repl;

\c echo;
CREATE PUBLICATION all_tables_pub FOR ALL TABLES;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO logical_repl;    
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO logical_repl;
```
4. verification
```
echo=# SELECT usename, usesuper, userepl FROM pg_user WHERE usename = 'logical_repl';
   usename    | usesuper | userepl
--------------+----------+---------
 logical_repl | f        | t
(1 row)

echo=# SELECT * FROM pg_publication;
  oid  |    pubname     | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate | pubviaroot
-------+----------------+----------+--------------+-----------+-----------+-----------+-------------+------------
 16390 | all_tables_pub |       10 | t            | t         | t         | t         | t           | f
(1 row)

echo=# SELECT * FROM pg_publication_tables;
 pubname | schemaname | tablename | attnames | rowfilter
---------+------------+-----------+----------+-----------
(0 rows)

echo=# \c echo logical_repl
You are now connected to database "echo" as user "logical_repl".
echo=> SELECT * FROM pg_publication;
  oid  |    pubname     | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate | pubviaroot
-------+----------------+----------+--------------+-----------+-----------+-----------+-------------+------------
 16390 | all_tables_pub |       10 | t            | t         | t         | t         | t           | f
(1 row)
```
5. run the deployment to the postgresql in deployment to create the table
```
kubectl create -f deployment-before-migrate.yaml
kubectl port-forward svc/deployment-migrate 8080:8080

echo=> \dt
         List of relations
 Schema | Name  | Type  |   Owner
--------+-------+-------+-----------
 public | echos | table | echo_user
(1 row)

echo=> select * from echos;
 created_at | updated_at | deleted_at | id | echo
------------+------------+------------+----+------
(0 rows)

curl http://localhost:8080/postgresql/test
58b152e8-8c96-4b3c-b9fb-194fdd3d7df4:test%

echo=> select * from echos;
          created_at           |          updated_at           | deleted_at |                  id                  | echo
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-11-09 08:59:58.212282+00 | 2025-11-09 08:59:58.210585+00 |            | 58b152e8-8c96-4b3c-b9fb-194fdd3d7df4 | test
```

### Create the Database with CNPG
1. create the manifests
```
kubectl apply -f cluster.yaml
```
2. the first thing that the CNPG do it will actually import the schema from old database from .spec.bootstrap and .spec.externalClusters
```
spec:
  externalClusters:
  - name: legacy-postgresql-publisher
    connectionParameters:
      host: postgresql-publisher.cnpg-system.svc.cluster.local
      user: logical_repl
      dbname: echo
    password:
      name: cnpg-postgresql-publisher-creds
      key: password
  bootstrap:
    initdb:
      import:
        type: microservice
        schemaOnly: true
        databases:
          - echo
        source:
          externalCluster: legacy-postgresql-publisher
```
externalClusters will be used to configure access to one or more PostgreSQL clusters as sources. in this case, the external cluster is out database outside of CNPG, can be VM, or any type of PostgreSQL database.

https://cloudnative-pg.io/documentation/1.26/bootstrap/#the-externalclusters-section


Bootstrap .initdb.import.schemaOnly set as true will tell CNPG to import the schema of database echo from the externalCluster defined. which mean when the CNPG cluster is created, the first thing it will do it importing the schema of database echo

https://cloudnative-pg.io/documentation/1.26/database_import/#the-microservice-type

you can follow this command to validate
```
k get pod | grep import
cnpg-postgresql-publisher-1-import-6b8ch       0/1     Init:0/1   0              10s

k logs cnpg-postgresql-publisher-1-import-6b8ch

{"level":"info","ts":"2025-11-09T13:34:02.310198507Z","msg":"dropping user-defined extensions from the target (empty) database","logging_pod":"cnpg-postgresql-publisher-1-import"}
{"level":"info","ts":"2025-11-09T13:34:02.315041387Z","msg":"temporarily granting superuser permission to owner user","logging_pod":"cnpg-postgresql-publisher-1-import","owner":"app"}
{"level":"info","ts":"2025-11-09T13:34:02.317266806Z","msg":"executing database importing section","logging_pod":"cnpg-postgresql-publisher-1-import","databaseName":"echo","section":"pre-data"}
{"level":"info","ts":"2025-11-09T13:34:02.317278636Z","msg":"Running pg_restore","logging_pod":"cnpg-postgresql-publisher-1-import","cmd":"pg_restore","options":["-U","postgres","--no-owner","--no-privileges","--role=app","-d","app","--section","pre-data","/var/lib/postgresql/data/pgdata/dumps/echo.dump"]}
```
3. it will start the new database master replica
```
k get pod

cnpg-postgresql-publisher-1                          2/2     Running   0              3m7s
cnpg-postgresql-publisher-2                          2/2     Running   0              73s

k get cluster
NAME                        AGE     INSTANCES   READY   STATUS                     PRIMARY
cnpg-postgresql-publisher   8m33s   2           2       Cluster in healthy state   cnpg-postgresql-publisher-1

```
4. let's verify the schema before starting the subscriber
```
kubectl port-forward svc/cnpg-postgresql-publisher-rw -n cnpg-system 5432:5432
export PG_PASSWORD=$(kubectl get secret cnpg-postgresql-publisher-app -o jsonpath='{.data.password}' | base64 -d)

psql -h 127.0.0.1 -p 5432 -d app -U app

psql (18.0, server 17.5 (Debian 17.5-1.pgdg110+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

app=> \dt
         List of tables
 Schema | Name  | Type  | Owner 
--------+-------+-------+-------
 public | echos | table | app
(1 row)

app=> \d+ echos
                                                     Table "public.echos"
   Column   |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 created_at | timestamp with time zone |           |          |         | plain    |             |              | 
 updated_at | timestamp with time zone |           |          |         | plain    |             |              | 
 deleted_at | timestamp with time zone |           |          |         | plain    |             |              | 
 id         | character varying(255)   |           | not null |         | extended |             |              | 
 echo       | character varying(255)   |           | not null |         | extended |             |              | 
Indexes:
    "echos_pkey" PRIMARY KEY, btree (id)
    "idx_echos_deleted_at" btree (deleted_at)
Publications:
    "all_tables_pub"
Access method: heap
```

5. verify the schema on old database
```
k exec -it postgresql-publisher-786dfc64f7-k6zdv -- bash

root@postgresql-publisher-786dfc64f7-k6zdv:/# psql -U echo_user -d echo
psql (15.14 (Debian 15.14-1.pgdg13+1))
Type "help" for help.

echo=# \d+ echos
                                                     Table "public.echos"
   Column   |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 created_at | timestamp with time zone |           |          |         | plain    |             |              | 
 updated_at | timestamp with time zone |           |          |         | plain    |             |              | 
 deleted_at | timestamp with time zone |           |          |         | plain    |             |              | 
 id         | character varying(255)   |           | not null |         | extended |             |              | 
 echo       | character varying(255)   |           | not null |         | extended |             |              | 
Indexes:
    "echos_pkey" PRIMARY KEY, btree (id)
    "idx_echos_deleted_at" btree (deleted_at)
Publications:
    "all_tables_pub"
Access method: heap
```

6. if you check, the schema is already same, but if we check the new database, there are no entry yet
```
app=> select * from echos;
 created_at | updated_at | deleted_at | id | echo 
------------+------------+------------+----+------
(0 rows)
```

### Create subscriber
To start sync the data from old database to CNPG database, create subscriber object
```
apiVersion: postgresql.cnpg.io/v1
kind: Subscription
metadata:
  name: cnpg-postgresql-publisher
  namespace: cnpg-system
spec:
  cluster:
    name: cnpg-postgresql-publisher
  dbname: app
  name: subscriber
  externalClusterName: legacy-postgresql-publisher
  publicationName: all_tables_pub
```
.spec.cluster.name is the CNPG cluster name where the subscriber will be created. it will be created with name spec.name from publisher defined on the externalClusterName (see the Cluster object). In the preparation, we create publisher with name all_tables_pub

spec.dbname is the database name of the CNPG postgresql. in this case, since we are not set the database name, the default will be app

1. apply the subscription object
```
kubectl apply -f subscriber.yaml

k get subscription
NAME                        AGE   CLUSTER                     PG NAME      APPLIED   MESSAGE
cnpg-postgresql-publisher   55s   cnpg-postgresql-publisher   subscriber   true
```

2. check the database on the CNPG postgresql
```
app=> select * from echos;
          created_at           |          updated_at           | deleted_at |                  id                  | echo 
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-11-09 08:59:58.212282+00 | 2025-11-09 08:59:58.210585+00 |            | 58b152e8-8c96-4b3c-b9fb-194fdd3d7df4 | test
```
the records is now sync with the old/legacy database.

3. let's try to call service that still on the old database
```
kubectl port-forward svc/deployment-migrate 8080:8080

curl http://localhost:8080/postgresql/test
f56f9609-1542-4549-8c1d-f141ff7dc7df:test%
```

4. it will replicated directly to CNPG cluster
```
app=> select * from echos;
          created_at           |          updated_at           | deleted_at |                  id                  | echo 
-------------------------------+-------------------------------+------------+--------------------------------------+------
 2025-11-09 08:59:58.212282+00 | 2025-11-09 08:59:58.210585+00 |            | 58b152e8-8c96-4b3c-b9fb-194fdd3d7df4 | test
 2025-11-09 14:11:15.477467+00 | 2025-11-09 14:11:15.474096+00 |            | f56f9609-1542-4549-8c1d-f141ff7dc7df | test
(2 rows)
```

### Next Step
1. Plan deployment freeze where you stop all traffic
2. change your database configuration to new CNPG database
3. remove the subscription CRDs
4. completely deprecated your old/legacy database on VMs!

### Source
https://www.gabrielebartolini.it/articles/2024/03/cloudnativepg-recipe-5-how-to-migrate-your-postgresql-database-in-kubernetes-with-~0-downtime-from-anywhere/
https://cloudnative-pg.io/documentation/1.26/bootstrap/#bootstrap-from-another-cluster