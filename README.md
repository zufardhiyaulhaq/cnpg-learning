# CloudNativePG Learning

## Learning
1. always implement primaryUpdateStrategy & primaryUpdateMethod. this to avoid master pod being deleted for any activity (upgrade, adding wal storage, etc). switchover is prefered since it's faster instead of waiting Kubernetes master pod to shutdown and start back.
```
spec:
  primaryUpdateStrategy: unsupervised
  primaryUpdateMethod: switchover
```
2. split WAL storage to speed up I/O performance and you can fine tune the storageClass & performance for PGDATA and WAL.
```
  storage:
    size: 20Gi
    storageClass: gtf-ack-essd-pl0-wait
  walStorage:
    size: 1Gi
    storageClass: gtf-ack-essd-pl0-wait
```
3. always check the image is exist to prevent any distruption during activity (issue during activity is more hard to fix). https://github.com/cloudnative-pg/postgres-containers/pkgs/container/postgresql
4. always plan which storageclass to choose and make sure it's support online volume resizing for easier increase of disk size
5. always plan to which size during disk increase since reducing disk size is not possible.
6. Activity that required master switchover/restart
   1. adding WAL Storage
   2. Postgresql Upgrade
   3. Change StorageClass
   4. Change resource request & limit spcification
   5. Implementing WAL archival
   6. Upgrade CNPG Operator (switchover/restart all Postgresql Cluster!)
7. Activity that not required master switchover/restart
   1. modify synchronous replica
   2. increase disk
   3. set postgresql parameters archive_timeout
   4. adding subscriber & publisher
   5. adding users/roles

8. Replication slot are enable by default https://cloudnative-pg.io/documentation/1.26/replication/#replication-slots
9. After major upgrade, the existing base backup & WAL archive is only available for old the previous postgresql version.
10. Always install kubectl cpng plugin, it's really helpful for troubleshooting https://cloudnative-pg.io/documentation/1.26/kubectl-plugin/
11. when creating a new role in CNPG, make sure to align the role name on .spec.managed.roles[x].name with the basic-auth secret on key username. if you use different username, it will not work and not returning any error.
12. always test upgrading operator in test cluster and observe the behaviour. there is some bugs where ENABLE_INSTANCE_MANAGER_INPLACE_UPDATES is set to true but still rollout restart the cluster
13. always set primaryUpdateStrategy to supervised for all CNPG cluster before upgrading operator, preventing current master to switchover or restarted if something weird happen.
```
spec:
  primaryUpdateStrategy: supervised
```

## Cheatsheet

### Switchover master
```
kubectl get pod
kubectl get clusters.postgresql.cnpg.io

kubectl cnpg promote <cluster-name> <replica-pod-to-be-promoted>
```

### Execute command without port-forward
```
kubectl cnpg psql echo-postgresql -- -qAt -c 'SELECT version()'
```

### Check Synchronous Replica
```
app=> SHOW synchronous_commit;
app=> SHOW synchronous_standby_names;
```

### Check Archival Timeout
```
app=> SHOW archive_timeout;
```

### Check replication slot
```
kubectl cnpg psql echo-postgresql -- -qAt -c 'SELECT * FROM pg_replication_slots;'
kubectl cnpg psql echo-postgresql -- -qAt -c 'SELECT slot_name, active, restart_lsn FROM pg_replication_slots'
```

### Check pg_hba.conf
```
kubectl cnpg psql echo-postgresql -- -qAt -c 'SELECT * FROM pg_hba_file_rules;'
```

### Check Publisher configuration
```
 app=> SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete, pubtruncate, pubviaroot                                                                        FROM pg_publication;
            pubname             | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate | pubviaroot 
--------------------------------+--------------+-----------+-----------+-----------+-------------+------------
 echo-postgresql-16-9-publisher | t            | t         | t         | t         | t           | f
```

### Check roles with replication permission
```
 app=> SELECT rolname FROM pg_roles WHERE rolreplication = true;
```

### Check overall status of postgresql cluster
```
kubectl cnpg status echo-postgresql

Cluster Summary
Name                 cnpg-system/echo-postgresql
System ID:           7520503366892757021
PostgreSQL Image:    ghcr.io/cloudnative-pg/postgresql:17.5
Primary instance:    echo-postgresql-15
Primary start time:  2025-07-24 23:42:56 +0000 UTC (uptime 57h22m7s)
Status:              Cluster in healthy state 
Instances:           3
Ready instances:     3
Size:                670M
Current Write LSN:   0/82000000 (Timeline: 8 - WAL File: 000000080000000000000082)

Continuous Backup status
First Point of Recoverability:  2025-07-18T00:05:32Z
Working WAL archiving:          OK
WALs waiting to be archived:    0
Last Archived WAL:              000000080000000000000081   @   2025-07-27T08:36:44.940042Z
Last Failed WAL:                00000008.history           @   2025-07-24T23:42:55.962973Z

Streaming Replication status
Replication Slots Enabled
Name                Sent LSN    Write LSN   Flush LSN   Replay LSN  Write Lag  Flush Lag  Replay Lag  State      Sync State  Sync Priority  Replication Slot
----                --------    ---------   ---------   ----------  ---------  ---------  ----------  -----      ----------  -------------  ----------------
echo-postgresql-13  0/82000000  0/82000000  0/82000000  0/82000000  00:00:00   00:00:00   00:00:00    streaming  quorum      1              active
echo-postgresql-16  0/82000000  0/82000000  0/82000000  0/82000000  00:00:00   00:00:00   00:00:00    streaming  quorum      1              active

Instances status
Name                Current LSN  Replication role  Status  QoS        Manager Version  Node
----                -----------  ----------------  ------  ---        ---------------  ----
echo-postgresql-15  0/82000000   Primary           OK      Burstable  1.26.0           ap-southeast-5.10.195.167.67
echo-postgresql-13  0/82000000   Standby (sync)    OK      Burstable  1.26.0           ap-southeast-5.10.195.169.204
echo-postgresql-16  0/82000000   Standby (sync)    OK      Burstable  1.26.0           ap-southeast-5.10.195.167.252

Plugins status
Name                            Version  Status  Reported Operator Capabilities
----                            -------  ------  ------------------------------
barman-cloud.cloudnative-pg.io  0.5.0    N/A     Reconciler Hooks, Lifecycle Service
```
## Troubleshooting

### Replica cannot join master - Error Waiting for the instances to become active

if replica is missing from replication slot and it's not active
```
kubectl get pod

echo-postgresql-13                              2/2     Running            0               2d7h
echo-postgresql-14                              1/2     Running            28 (20m ago)    28h
echo-postgresql-15                              2/2     Running            0               19d

kubectl cnpg psql echo-postgresql -- -qAt -c 'SELECT slot_name, active, restart_lsn FROM pg_replication_slots'
_cnpg_echo_postgresql_14|f|0/7A0000A0
_cnpg_echo_postgresql_13|t|0/80000000
```

the easiest way is to completely remove the PVC, PV, and pod.
```
k delete pv d-k1a6j280bbsxtsbcnyd6 d-k1a19hs0finsu6ev5e9d
persistentvolume "d-k1a6j280bbsxtsbcnyd6" deleted
persistentvolume "d-k1a19hs0finsu6ev5e9d" deleted

k delete pvc echo-postgresql-14-wal echo-postgresql-14
persistentvolumeclaim "echo-postgresql-14-wal" deleted
persistentvolumeclaim "echo-postgresql-14" deleted

k delete pod echo-postgresql-14
```

or you can use cnpg plugin
```
kubectl cnpg destroy echo-postgresql 14
```

cloudnative-pg will automatically create a new pod with higher number
```
echo-postgresql-13                              2/2     Running            0               2d7h
echo-postgresql-15                              2/2     Running            0               19d
echo-postgresql-16                              2/2     Running            0               3m40s
```

### Pod stuck in Terminating during major upgrade

just force delete the old pod
```
k delete pod echo-postgresql-major-upgrade-1 --force
```

### Open Questions
1. it's possible to separate replica with master storage size
2. it's possible to downgrade the disk