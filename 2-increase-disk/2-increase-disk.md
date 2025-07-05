# Increase Disk

1. port forward service & test curl
```
kubectl port-forward svc/echo-postgresql 8080:8080
while true; do curl http://localhost:8080/postgresql/test; echo ""; sleep 0.1; done
```
2. updating disk postgresql cluster from 1Gi to 2Gi
```
kubectl apply -f cluster.yaml
```
3. there is no downtime on the clusters
```
zufar.dhiyaullhaq@MacBookPro ~ % k get cluster
NAME              AGE     INSTANCES   READY   STATUS                     PRIMARY
echo-postgresql   7d21h   4           4       Cluster in healthy state   echo-postgresql-2
```
4. disk seems updated seamlessly
```
zufar.dhiyaullhaq@MacBookPro 2-increase-disk % k get pvc
NAME                STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS            VOLUMEATTRIBUTESCLASS   AGE
echo-postgresql-1   Bound    d-k1aixukc38i26yta2awf   2Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
echo-postgresql-2   Bound    d-k1abj0np1naq984llm6a   2Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
echo-postgresql-3   Bound    d-k1a3dqbtva9i212x5jjl   2Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
echo-postgresql-4   Bound    d-k1acdss6owu4m8amb3ch   1Gi        RWO            gtf-ack-essd-pl0-wait   <unset>                 7d21h
```
5. there is process of expanding volume
```
Events:
  Type     Reason                      Age   From                                              Message
  ----     ------                      ----  ----                                              -------
  Warning  ExternalExpanding           110s  volume_expand                                     waiting for an external controller to expand this PVC
  Normal   Resizing                    110s  external-resizer diskplugin.csi.alibabacloud.com  External resizer is resizing volume d-k1abj0np1naq984llm6a
  Normal   FileSystemResizeRequired    107s  external-resizer diskplugin.csi.alibabacloud.com  Require file system resize of volume on node
  Normal   FileSystemResizeSuccessful  102s  kubelet                                           MountVolume.NodeExpandVolume succeeded for volume "d-k1abj0np1naq984llm6a" ap-southeast-5.10.195.169.24
```