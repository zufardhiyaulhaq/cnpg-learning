# Increase Disk

1. port forward service & test curl
```
kubectl port-forward svc/echo-postgresql 8080:8080
while true; do curl http://localhost:8080/postgresql/test; echo ""; sleep 0.1; done
```
2. change storageclass of cluster
```
kubectl apply -f cluster.yaml
```
3. storageclass for existing replica will not change automatically, if new replica is created, the new pod will start using the new storageclass
```
zufar.dhiyaullhaq@MacBookPro 3-change-storageclass % k get pvc              
NAME                STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS            VOLUMEATTRIBUTESCLASS   AGE
echo-postgresql-1   Bound    d-k1aixukc38i26yta2awf   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
echo-postgresql-2   Bound    d-k1abj0np1naq984llm6a   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
echo-postgresql-3   Bound    d-k1a3dqbtva9i212x5jjl   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
echo-postgresql-4   Bound    d-k1acdss6owu4m8amb3ch   20Gi       RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
echo-postgresql-8   Bound    d-k1acpik2wwsu44oks407   20Gi       RWO            gtf-ack-essd-pl1-wait   <unset>                 29s

zufar.dhiyaullhaq@MacBookPro 3-change-storageclass % k get pod
NAME                                   READY   STATUS    RESTARTS   AGE
echo-postgresql-1                      1/1     Running   0          12h
echo-postgresql-2                      1/1     Running   0          3d22h
echo-postgresql-3                      1/1     Running   0          7d21h
echo-postgresql-4                      1/1     Running   0          7d21h
echo-postgresql-8                      1/1     Running   0          66s
```
4. promote new pod as master, there will be small downtime
```
zufar.dhiyaullhaq@MacBookPro 3-change-storageclass % kubectl cnpg promote echo-postgresql echo-postgresql-8
{"level":"info","ts":"2025-07-05T10:42:56.261724+07:00","msg":"Cluster has become unhealthy"}
Node echo-postgresql-8 in cluster echo-postgresql will be promoted

zufar.dhiyaullhaq@MacBookPro 3-change-storageclass % kubectl get cluster                                                                  
NAME              AGE     INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql   7d21h   5           5       Cluster in healthy state   echo-postgresql-8
```
5. delete the replica 1 by 1
```
kubectl delete pvc/echo-postgresql-1 pod/echo-postgresql-1
kubectl delete pvc/echo-postgresql-2 pod/echo-postgresql-2
kubectl delete pvc/echo-postgresql-3 pod/echo-postgresql-3
kubectl delete pvc/echo-postgresql-4 pod/echo-postgresql-4
```
6. new PVC & Pods will be spawn without downtime
```
zufar.dhiyaullhaq@MacBookPro ~ % k get pvc
NAME                 STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS            VOLUMEATTRIBUTESCLASS   AGE
echo-postgresql-10   Bound    d-k1acrrpgby4mh7nzv72u   20Gi       RWO            gtf-ack-essd-pl1-wait   <unset>                 3m33s
echo-postgresql-11   Bound    d-k1a693vbat805shp01pz   20Gi       RWO            gtf-ack-essd-pl1-wait   <unset>                 2m38s
echo-postgresql-12   Bound    d-k1a4huvaxxks2tuov4co   20Gi       RWO            gtf-ack-essd-pl1-wait   <unset>                 65s
echo-postgresql-8    Bound    d-k1acpik2wwsu44oks407   20Gi       RWO            gtf-ack-essd-pl1-wait   <unset>                 9m25s
echo-postgresql-9    Bound    d-k1a3qecdmzap8hqf45hf   20Gi       RWO            gtf-ack-essd-pl1-wait   <unset>                 4m42s

echo-postgresql-10                     1/1     Running   0          2m55s
echo-postgresql-11                     1/1     Running   0          99s
echo-postgresql-12                     1/1     Running   0          27s
echo-postgresql-8                      1/1     Running   0          8m46s
echo-postgresql-9                      1/1     Running   0          4m7s
```

https://cloudnative-pg.io/documentation/1.26/storage/
https://cloudnative-pg.io/documentation/1.26/kubectl-plugin/
https://cloudnative-pg.io/documentation/1.26/storage/#re-creating-storage

## Issues

### PVC creation Stuck during new Replica
when creating a new replica with new storageclass, if PVC cannot be created with new storageclass because of some reason (new storageclass expect minimal capacity, etc)
1. revert the cluster manifest & replica
2. delete the PVC
3. delete the new pod
4. delete the job for new pod
   