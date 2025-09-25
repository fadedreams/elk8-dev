# ELK8-Dev Helm Chart
A fast, resource-efficient ELK stack, derived from the Elastic Helm repository[](https://helm.elastic.co) for development-purpose. Authentication is disabled, and persistent volumes are enabled for Elasticsearch and Kibana.

## Prerequisites
Kubernetes cluster (e.g., Minikube, Kind, or a managed cluster like EKS/GKE/AKS).
Helm 3.x installed.

## Installation
## Method 1: Install via GitHub Repository (Recommended)
Clone the repository and install individual components using the official Elastic Helm charts with custom values files.
```
helm repo add elastic https://helm.elastic.co
helm repo update

# Add Elastic Helm repository
# Clone the repository
git clone https://github.com/fadedreams/elk8-dev.git
cd elk8-dev

# Install components
helm install elk-elasticsearch elastic/elasticsearch -f ./k8s/elasticsearch-values.yaml --namespace logging --create-namespace
helm install elk-kibana elastic/kibana -f ./k8s/kibana-values.yaml --namespace logging
helm install elk-logstash elastic/logstash -f ./k8s/logstash-values.yaml --namespace logging
helm install elk-filebeat elastic/filebeat -f ./k8s/filebeat-values.yaml --namespace logging
```

### Useful Commands
#### Retrieve Elasticsearch user password
```
ELASTIC_PASSWORD=$(kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d)
echo "$ELASTIC_PASSWORD"
```
#### List Elasticsearch indices
```
curl -u elastic:$ELASTIC_PASSWORD http://localhost:9200/_cat/indices?v
```
#### Port Forwarding
```
kubectl port-forward service/elasticsearch-master -n logging 9200:9200 &
kubectl port-forward service/kibana -n logging 5601:5601               &
kubectl port-forward service/logstash -n logging 5044:5044 8080:8080   &
```

#### Monitor Logstash/Filebeat logs
```
kubectl logs logstash-0 -n logging -f | grep -E "beats|elasticsearch|event|error"
kubectl logs filebeat-qr2rv -n logging -f | grep -E "logstash|error|publish|beats"
```
#### Querying Elasticsearch
```
curl -k -u elastic:$ELASTIC_PASSWORD "https://localhost:9200/logstash-*/_search?pretty" -H 'Content-Type: application/json' -d '{
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1h" } } },
      ]
    }
  },
  "size": 10
}'
```



## Method 2: Install via ArtifactHub Repository (Not Recommended)
Note: This method is not recommended as it may fail under certain conditions
```
helm repo add fadedreams https://artifacthub.io/storage/helm-charts/fadedreams
helm repo update
helm install my-elk fadedreams/elk8-dev --set elasticsearch.enabled=true --namespace logging --create-namespace
helm install my-elk fadedreams/elk8-dev --set kibana.enabled=true --namespace logging --create-namespace
helm install my-elk fadedreams/elk8-dev --set logstash.enabled=true --namespace logging --create-namespace
helm install my-elk fadedreams/elk8-dev --set filebeat.enabled=true --namespace logging --create-namespace
```
