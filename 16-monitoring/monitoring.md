 # Monitoring

CloudNativePG provide default metrics defined in https://cloudnative-pg.io/documentation/1.26/monitoring/. the operator recommend us to create PodMonitor for each of the cluster. The operator is also provide build in grafana that we can use in https://github.com/cloudnative-pg/grafana-dashboards/blob/main/charts/cluster/grafana-dashboard.json.

1. Check metrics is exist in the clusters
```
k get pod | grep echo-postgresql                 
echo-postgresql-1                               2/2     Running   0              4d1h
echo-postgresql-2                               2/2     Running   0              4d1h
echo-postgresql-3                               2/2     Running   0              4d1h

k port-forward pod/echo-postgresql-1 9187:9187
```

2. get the metrics
```
curl http://localhost:9187/metrics
# HELP cnpg_backends_max_tx_duration_seconds Maximum duration of a transaction in seconds
# TYPE cnpg_backends_max_tx_duration_seconds gauge
cnpg_backends_max_tx_duration_seconds{application_name="",datname="app",state="idle",usename="app"} 0
cnpg_backends_max_tx_duration_seconds{application_name="cnpg_metrics_exporter",datname="app",state="active",usename="postgres"} 0
cnpg_backends_max_tx_duration_seconds{application_name="echo-postgresql-2",datname="",state="active",usename="streaming_replica"} 0
cnpg_backends_max_tx_duration_seconds{application_name="echo-postgresql-3",datname="",state="active",usename="streaming_replica"} 0
# HELP cnpg_backends_total Number of backends
# TYPE cnpg_backends_total gauge
cnpg_backends_total{application_name="",datname="app",state="idle",usename="app"} 1
cnpg_backends_total{application_name="cnpg_metrics_exporter",datname="app",state="active",usename="postgres"} 1
cnpg_backends_total{application_name="echo-postgresql-2",datname="",state="active",usename="streaming_replica"} 1
cnpg_backends_total{application_name="echo-postgresql-3",datname="",state="active",usename="streaming_replica"} 1
# HELP cnpg_backends_waiting_total Total number of backends that are currently waiting on other queries
# TYPE cnpg_backends_waiting_total gauge
cnpg_backends_waiting_total 0
# HELP cnpg_collector_collection_duration_seconds Collection time duration in seconds
# TYPE cnpg_collector_collection_duration_seconds gauge
cnpg_collector_collection_duration_seconds{collector="Collect.cnpg"} 0.201331799
cnpg_collector_collection_duration_seconds{collector="Collect.up"} 0.002624492
```

3. create PodMonitor to scrap the metrics. each Pod have port called metrics, which listen in port 9187
```
kubectl apply -f podmonitor.yaml
```

4. make sure metrics is exposed in grafana and you can start importing the grafana dashboard from https://github.com/cloudnative-pg/grafana-dashboards/blob/main/charts/cluster/grafana-dashboard.json

the default metrics and dashboard have provide lot of data for us to troubleshoot:
- WAL & Backup metrics
- Replication metrics
- storage metrics
- lag metrics
- connections metrics
- configuration and extention metric
