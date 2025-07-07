# Change Request limit of Postgresql

1. apply the request limit
```
kubectl apply -f cluster.yaml

apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: echo-postgresql
  namespace: cnpg-system
spec:
  resources:
    requests:
      cpu: 100m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi
```
2. Replica will be restarted
```
$ kubectl get cluster
NAME                                   AGE    INSTANCES   READY   STATUS                                       PRIMARY
echo-postgresql                        10d    3           2       Waiting for the instances to become active   echo-postgresql-13

kubectl get pod
echo-postgresql-13                         1/1     Running    0          2d3h
echo-postgresql-14                         0/1     Init:0/1   0          17s
echo-postgresql-15                         1/1     Running    0          58s
```
3. Since we use unsupervised with switchover, primary will switchover automatically
```
zufar.dhiyaullhaq@MacBookPro cnpg-learning % k get cluster                                                                                            
NAME                                   AGE    INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql                        10d    3           3       Cluster in healthy state   echo-postgresql-14

k get pod echo-postgresql-14 -oyaml | grep resources -A 10
--
    resources:
      limits:
        cpu: 500m
        memory: 1Gi
      requests:
        cpu: 100m
        memory: 512Mi
```

https://cloudnative-pg.io/documentation/1.26/cloudnative-pg.v1/#postgresql-cnpg-io-v1-ClusterSpec