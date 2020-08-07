# Connect to CCloud
Using Confluent Operator to deploy a self-managed Kafka Connect cluster linked to Confluent Cloud.

Last updated: 08-07-20.

Tested on: Minikube, Azure Kubernetes (AKS), Google Kubernetes Engine (GKE)

k8s version: 1.16.10

Contact: apowell@confluent.io

## STEPS:
## Build Docker image for Kafka Connect source/sink connectors. See [these instructions](./docker-image/CONNECT_IMAGE.md) for full steps.
### *(You may skip this step if you are able to use one of the connectors packaged inside cp-server-connect-operator)*

```
#Check/change contents of docker-image/Dockerfile for connectors you wish to add
#Build the image:
docker build -t alecpowell18/connect-custom:0.1 .
docker push alecpowell18/connect-custom:0.1
```
Publish to a place / repo where your k8s cluster will be able to pull it.


## Deploy Operator Helm Chart

*(Pre-req: spin up a k8s cluster)*

Download Operator
```
wget https://platform-ops-bin.s3-us-west-1.amazonaws.com/operator/confluent-operator-5.5.1.tar.gz
tar -xvf confluent-operator-5.5.1.tar.gz
cd confluent-operator/
```

Create new values.yaml file for your deployment
```
cp helm/providers/azure.yaml values-operator.yaml
vim values-operator.yaml  # update license key, region and zones, enable&configure LB for external access (if desired)
```

Change # replicas, docker image name (to refer to image you just built), for the Connect cluster in values-operator.yaml
Also, make sure to add API Key and API Secret for access to Confluent Cloud inside global.sasl.username / password.
```
vim values-operator.yaml
```

Create a namespace for Confluent platform inside your k8s cluster.
```
kubectl create namespace confluent
kubectl config set-context --current --namespace confluent
```

Deploy Operator, using path to confluent-operator directory inside /helm/ directory to pick up the Helm charts.
```
helm install operator ./helm/confluent-operator --namespace confluent --values values-operator.yaml --set operator.enabled=true
```

## Deploy the Connect cluster

Step 1: Create your API Keys and Service Account inside Confluent Cloud: https://docs.confluent.io/current/cloud/access-management/service-account.html

Step 2: Make sure that dependency for Kafka cluster is correctly linked to your Confluent Cloud cluster and API key inside values.yaml in `connect` section:
```
dependencies:
  kafka:
    bootstrapEndpoint: pkc-abcde.eastus.azure.confluent.cloud:9092
    brokerCount: 3
    username: <API_KEY>
    password: <API_KEY_SECRET>
    tls:
      enabled: true
      internal: true
      authentication:
        type: "plain"
  schemaRegistry:
    enabled: false
    url: ""
```
Step 3: Deploy the Helm chart.
```
helm install connect ./helm/confluent-operator --namespace confluent --values values-operator.yaml --set connect.enabled=true
```

## Deploy Confluent Control Center
Note: Per Confluent Cloud requirements, one configuration inside the C3 Helm chart must be updated.

Edit the PSC in confluent-operator/charts/controlcenter/templates/controlcenter-psc.yml to include the config:
> confluent.metrics.topic.max.message.bytes=8388608

Then, inside values-operator.yaml, make sure that the dependency for Kafka cluster is correctly linked to your Confluent Cloud cluster and API key in `controlcenter` section:
```
dependencies:
  c3KafkaCluster:
    brokerCount: 3
    bootstrapEndpoint: pkc-abcde.eastus.azure.confluent.cloud:9092
    zookeeper:
      endpoint: zookeeper:2181
    tls:
      enabled: true
      internal: true
      authentication:
        type: "plain"
  connectCluster:
    enabled: true
    url: http://connectors:8083
  # if self-managing ksqlDB, you may set to true
  ksql:
    enabled: false
    url: http://ksql:9088
  schemaRegistry:
    enabled: false
    url: http://schemaregistry:8081
```

Then deploy the Helm chart.
```
helm install controlcenter ./helm/confluent-operator --namespace confluent --values values-operator.yaml --set controlcenter.enabled=true
```

## Confirm the k8s pods are spun up and ready.
```
kubectl get pods -n confluent
kubectl get services -n confluent
```

## Start your connector(s)

You can either deploy through GUI at Confluent Control Center (C3), or by exposing the REST API for Connect locally.

#### A) Deploy through C3

Port forward the Control Center to your localhost (in this case, forwards all traffic to localhost:12345):
```
kubectl -n confluent port-forward controlcenter-0 12345:9021
```

Deploy connectors through the attached Connect cluster in GUI, by navigating your browser to port 12345.
```
open http://localhost:12345  # username / pw = admin / Developer1
```

#### B) Deploy through Kafka Connect's REST API

Port forward locally, so that you can publish to the Connect REST API
```
kubectl -n confluent port-forward connectors 8083:8083
```
Get the list of available connectors
```
curl localhost:8083/connector-plugins/ | jq
```
Get the running connectors
```
curl localhost:8083/connectors/ | jq
```
*Per datagen-users.config, CREATE the Kafka topic first in the CCloud GUI.
Or , from the CCloud CLI:*
```
ccloud kafka topic create users
```
Start the datagen connector
```
curl -H "Content-Type: application/json" --data @connectors-config/datagen-users.config localhost:8083/connectors
```
Check for "RUNNING" status
```
curl localhost:8083/connectors/datagen-users/status | jq
```
Check CCloud GUI to see that messages are being produced to your topic.

After few mins, delete the connector to stop producing.
```
curl -X DELETE http://localhost:8083/connectors/datagen-users
```

## Cleanup
```
helm del controlcenter
helm del connect
helm del operator
```

### Additional notes:

This model can be extended to ksqlDB using the `ksql` configuration inside `values.yaml`. However, note the following configs which must be overwritten in the PSC in confluent-operator/charts/ksql/templates/ksql-psc.yml:
```
ksql.internal.topic.replicas=3
ksql.streams.replication.factor=3
ksql.logging.processing.topic.replication.factor=3
```
