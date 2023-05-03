# Deploying an OpenSearch Cluster for Maken

## Pre-requisites

Before you can begin, you need to have [`gcloud`](https://cloud.google.com/sdk/docs/install), [`kubectl`](https://kubernetes.io/docs/tasks/tools/), and [`helm`](https://helm.sh/docs/intro/install/) installed, plus a Google Cloud project in which you are authenticated.

## Steps

1. Crete a Kubernetes cluster and get the credentials

```bash
export CLUSTER_NAME=maken-cluster
export CLUSTER_ZONE=europe-north1-a
gcloud container clusters create $CLUSTER_NAME \
    --zone $CLUSTER_ZONE \
    --node-locations $CLUSTER_ZONE \
    --num-nodes 2 \
    --enable-autoscaling --min-nodes 2 --max-nodes 5
gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE
```

2. A `DaemonSet` is needed to ensure cluster nodes set a proper value for `vm.max_map_count` in order to get OpenDistro to work.

```bash
kubectl apply -f max_map_count.yaml
```

3. Provision a node pool with at least 2 nodes with 32GB RAM and 8 CPUs (`e2-standard-8`, recommended `n2-standard-16` for 1M vectors in 2 shard)

```bash
gcloud container node-pools create "$CLUSTER_NAME-pool" --cluster $CLUSTER_NAME \
    --machine-type e2-standard-8 --num-nodes 2 \
    --disk-size 500G --disk-type pd-ssd \
    --enable-autoscaling --min-nodes 2 --max-nodes 5 \
    --zone $CLUSTER_ZONE
```

And optionally delete the default node pool

```bash
gcloud container node-pools delete default-pool --cluster $CLUSTER_NAME --zone $CLUSTER_ZONE
```

4. Deploy OpenSearch with the custom `values.yaml`.

```bash
helm install maken-index --values=values.yaml opensearch/opensearch
```

5. Create a load balancer and get the IP (Ctrl-C when ready)

```bash
kubectl apply -f load_balancer.yaml
kubectl get service/maken-index-service --output jsonpath='{.status.loadBalancer.ingress[0].ip}' --watch
```

In order to expose OpenDistro outside the cluster, the property `networking.gke.io/load-balancer-type` in `load_balancer.yaml` needs to be set to `"External"` and redeployed. In that case, it is strongly recommended to add `loadBalancerSourceRanges` to filter by IPs.

6. Wait until internal load balancer is available and then create a config map.

```bash
export CLUSTER_INDEX_IP=$(kubectl get service/maken-index-service --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
kubectl create configmap maken-index-api-config \
    --from-literal ES_HOST=$CLUSTER_INDEX_IP \
    --from-literal ES_SCHEME=https
```

7. Deploy the API and get the IP (Ctrl-C when ready)

```bash
kubectl apply -f api.yaml
kubectl get service/maken-api-service --output jsonpath='{.status.loadBalancer.ingress[0].ip}' --watch
```

To be able to manage the index and ingest data, you would need to enable port forwarding locally to the internal load balancer of the index service.

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE
kubectl port-forward $(kubectl get pod --selector="app.kubernetes.io/component=opensearch-cluster-master,app.kubernetes.io/instance=maken-index,app.kubernetes.io/name=opensearch" --output jsonpath='{.items[0].metadata.name}') 8080:9200
```
