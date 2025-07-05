# Syncronous Replica

PostgreSQL streaming replication is asynchronous by default. If the primary server crashes then some transactions that were committed may not have been replicated to the standby server, causing data loss. The amount of data loss is proportional to the replication delay at the time of failover.

in cloudnativePG. you can specify Quorum or Priority based syncronous replica. This configuration will set quorum syncronous replica which will wait for any of replica to acknowledge before completing the transaction.
```
  postgresql:
    synchronous:
      method: any
      number: 1
      dataDurability: required
```
`dataDurability` set to `required` ensure zero data loss (RPO=0) but may reduce database availability during network disruptions or standby failures. Set `dataDurability` to `prefered` can cause data loss if all replica is unavailable and master is also unavailable.

1. Update Cluster with Synchronous Replica
```
kubectl apply -f cluster.yaml
```
2. check if it's already applied
```
kubectl port-forward svc/echo-postgresql-rw -n cnpg-system 5432:5432

export PG_DB=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.dbname}' | base64 -d)
export PG_USER=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.user}' | base64 -d)
export PG_PASSWORD=$(kubectl get secret echo-postgresql-app -o jsonpath='{.data.password}' | base64 -d)

psql -h 127.0.0.1 -p 5432 -d app -U app

app=> SHOW synchronous_standby_names;
                       synchronous_standby_names
------------------------------------------------------------------------
 ANY 1 ("echo-postgresql-14","echo-postgresql-15","echo-postgresql-13")
(1 row)

app=> SHOW synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

app=> SHOW wal_level;
 wal_level
-----------
 logical
(1 row)
```

https://cloudnative-pg.io/documentation/1.26/replication/#synchronous-replication