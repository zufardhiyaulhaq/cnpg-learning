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

### Source
https://www.gabrielebartolini.it/articles/2024/03/cloudnativepg-recipe-5-how-to-migrate-your-postgresql-database-in-kubernetes-with-~0-downtime-from-anywhere/
https://cloudnative-pg.io/documentation/1.26/bootstrap/#bootstrap-from-another-cluster