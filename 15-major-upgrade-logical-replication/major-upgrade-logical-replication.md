https://cloudnative-pg.io/documentation/1.26/cloudnative-pg.v1/#postgresql-cnpg-io-v1-RoleConfiguration
https://cloudnative-pg.io/documentation/1.26/declarative_role_management/
https://cloudnative-pg.io/documentation/1.26/logical_replication/#example-of-live-migration-and-major-postgres-upgrade-with-logical-replication

# Major Upgrade with Logical Replication
This guide will help you walkthrough on how to do major PostgreSQL version upgrade between CNPG clusters. 

### Preparing old version of Database
1. create cluster with version 16.9, we will also create different roles, user, and password for replication. the role name is logical_replication, where username and password use secret type basic auth.
```
---
apiVersion: v1
kind: Secret
metadata:
  name: echo-postgresql-16-9-logical-replication-creds
  namespace: cnpg-system
data:
  username: cmVwbF91c2VybmFtZQ==
  password: cmVwbF9wYXNzd29yZA==
type: kubernetes.io/basic-auth
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
spec:
  managed:
    roles:
      - name: logical_replication
        login: true
        replication: true
        passwordSecret:
          name: echo-postgresql-16-9-logical-replication-creds
```
2. check the status and check the managed roles
```
k get cluster
NAME                   AGE   INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql-16-9   20m   2           2       Cluster in healthy state   echo-postgresql-16-9-1

k describe cluster echo-postgresql-16-9
  Managed Roles Status:
    By Status:
      Not - Managed:
        app
      Reconciled:
        logical_replication
      Reserved:
        postgres
        streaming_replica
    Password Status:
      logical_replication:
        Transaction ID:  745
```