# ELK8-Dev Helm Chart

A fast, resource-efficient ELK stack (Elasticsearch, Kibana, Logstash, Filebeat 8.5.1), derived from the
[Elastic Helm repository](https://helm.elastic.co), for development use. Security is on (the default of the
Elastic charts); persistent volumes are enabled for Elasticsearch.

## Prerequisites
- A Kubernetes cluster (kind, minikube, or a managed cluster) with a default StorageClass.
- Helm 3.x.
- About 4 GB of free RAM and 2 CPU cores for the full stack.
- On the node (for kind: on the host): `sysctl -w vm.max_map_count=262144`

## Method 1: Install from the Elastic charts with this repo's values files
```
helm repo add elastic https://helm.elastic.co
helm repo update
git clone https://github.com/fadedreams/elk8-dev.git
cd elk8-dev

helm install elk-elasticsearch elastic/elasticsearch -f ./k8s/elasticsearch-values.yaml --namespace logging --create-namespace
helm install elk-kibana elastic/kibana -f ./k8s/kibana-values.yaml --namespace logging
helm install elk-logstash elastic/logstash -f ./k8s/logstash-values.yaml --namespace logging
helm install elk-filebeat elastic/filebeat -f ./k8s/filebeat-values.yaml --namespace logging
```

## Method 2: Install as one umbrella chart (Artifact Hub / Helm repository)
```
helm repo add elk8-dev https://fadedreams.github.io/elk8-dev
helm repo update

# Stage 1: Elasticsearch only. Kibana's pre-install hook needs a running Elasticsearch,
# so Kibana cannot be installed in the same step.
helm install elk elk8-dev/elk8-dev --namespace logging --create-namespace \
  --set kibana.enabled=false --set logstash.enabled=false --set filebeat.enabled=false \
  --wait --timeout 10m

# Stage 2: everything else (chart defaults enable all components)
helm upgrade elk elk8-dev/elk8-dev --namespace logging --wait --timeout 15m
```
Notes:
- Use one release name (`elk` above). Do not run `helm install` several times with the same name.
- Elasticsearch needs a few minutes to become ready (readiness probe waits ~2 minutes).
- The generated `elastic` password is kept across `helm upgrade`. To choose your own, pass
  `--set elasticsearch.secret.password=...` on install and on every upgrade.
- Resource limits in `values.yaml` are sized for a small dev node; raise them for real workloads.

## Useful commands
#### Retrieve the Elasticsearch `elastic` password
```
ELASTIC_PASSWORD=$(kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d)
echo "$ELASTIC_PASSWORD"
```
#### Port forwarding
```
kubectl port-forward service/elasticsearch-master -n logging 9200:9200 &
kubectl port-forward service/kibana -n logging 5601:5601 &
```
Log in to Kibana (http://localhost:5601) as `elastic` with the password above.

#### List indices (HTTPS, self-signed certificate)
```
curl -k -u elastic:$ELASTIC_PASSWORD "https://localhost:9200/_cat/indices?v"
```
#### Querying logs
Logstash 8 writes to the data stream `logs-generic-default` (not `logstash-*`):
```
curl -k -u elastic:$ELASTIC_PASSWORD "https://localhost:9200/logs-generic-default/_search?pretty" \
  -H 'Content-Type: application/json' -d '{
  "query": { "range": { "@timestamp": { "gte": "now-1h" } } },
  "size": 10
}'
```
In Kibana create a Data View for `logs-generic-default` (or `logs-*`).

#### Monitor Logstash / Filebeat
```
kubectl logs logstash-0 -n logging -f | grep -E "beats|elasticsearch|error"
kubectl logs -l app=filebeat -n logging -f | grep -E "logstash|error|publish"
```

## Troubleshooting
- Elasticsearch pod `0/1` for several minutes: normal on first start; check `kubectl describe pod elasticsearch-master-0 -n logging`.
- `OOMKilled`: raise the memory limit of that component in `values.yaml`.
- Pods `Pending`: check for a default StorageClass (`kubectl get storageclass`) and free node resources.
