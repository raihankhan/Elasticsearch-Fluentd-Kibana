# Elasticsearch-Fluentd-Kibana (EFK)

This is a simple for for deploying an Elasticsearch cluster (version 7.17.3) with Kibana in Kubernetes. A fluentd daemonSet will also be deployed that will collect logs from kubernetes containers and ship them to elasticsearch index. 

Create a namespace first. I am using demo namesapce here. 
```
$ kubectl create namespace demo
```

Create a secret containing credentials for Elasticsearch. Use `elastic` user as it has all the administrative priviledges. Create a new secret named `elastic-credentials` with `username=elastic` and `password=<password>`.

```
$ kubectl create secret generic elastic-credentials --from-literal=username=elastic --from-literal=password=vXVWD81ms2s6B56KVGQO -n demo
```

Get the secret yaml.

```
$ kubectl get secret -n demo elastic-credentials -oyaml
apiVersion: v1
data:
  password: dlhWV0Q4MW1zMnM2QjU2S1ZHUU8=
  username: ZWxhc3RpYw==
kind: Secret
metadata:
  creationTimestamp: "2023-01-18T05:23:51Z"
  name: elastic-credentials
  namespace: demo
  resourceVersion: "57700"
  uid: ddce7dfb-f165-453c-b8f3-677ceffb1e0c
type: Opaque
```

Now, deploy Elasticsearch using helm chart provided by elastic. Make sure you have provided proper version, environment variables for authentication fetched from the credential secret, required node roles, storage and number of replicas in the `elasticsearch-values.yaml` file configurations. 
```
$ helm upgrade --install elasticsearch \
                elastic/elasticsearch \
                --namespace demo \
                --version "7.17.3" \
                --values ./elasticsearch-values.yaml 
```

Get all the pods & services created by the helm chart.

```
$ kubectl get pods,services,secrets -n demo
NAME                         READY   STATUS    RESTARTS   AGE
pod/elasticsearch-master-0   1/1     Running   0          13m
pod/elasticsearch-master-1   1/1     Running   0          13m
pod/elasticsearch-master-2   1/1     Running   0          13m

NAME                                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
service/elasticsearch-master            NodePort    10.96.77.101   <none>        9200:30181/TCP,9300:30208/TCP   13m
service/elasticsearch-master-headless   ClusterIP   None           <none>        9200/TCP,9300/TCP               13m

NAME                                         TYPE                 DATA   AGE
secret/elastic-credentials                   Opaque               2      38m
secret/sh.helm.release.v1.elasticsearch.v1   helm.sh/release.v1   1      13m
```

Expose the headless service to your localhost port 9200 and test the cluster state.
```
$ kubectl port-forward service/elasticsearch-master-headless -n demo 9200
Forwarding from 127.0.0.1:9200 -> 9200
Forwarding from [::1]:9200 -> 9200
```
```
$ curl -XGET -k -u 'elastic:vXVWD81ms2s6B56KVGQO' "http://localhost:9200/_cluster/health?pretty"
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 2,
  "active_shards" : 4,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

Elasticsearch cluster state is green. Let's move forward to install Kibana. Deploy Kibana using helm chart provided by elastic. Install the chart will same version flag as elasticsearch. Kibana is most compatible with similar elasticserch version. Make sure you have provided Elasticsearch host address, environment variables for authentication fetched from the credential secret, and number of replicas in the `kibana-values.yaml` file configurations. I am using the headless service IP as my `elasticsearchHosts`.

```
helm upgrade --install kibana \
  elastic/kibana \
  --namespace demo \
  --version "7.17.3" \
  --values ./kibana-values.yaml
```

