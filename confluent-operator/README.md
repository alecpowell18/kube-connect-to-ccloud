# Connect to CCloud
Using Confluent Operator to deploy a self-managed Kafka Connect cluster linked to Confluent Cloud.

Last updated: 12-11-20.

Tested on: Minikube, Azure Kubernetes Service (AKS), Google Kubernetes Engine (GKE), Elastic Kubernetes Service (EKS)

k8s versions used: 1.16, 1.18

Contact: apowell@confluent.io

## STEPS:
## 1) Build Docker image for Kafka Connect source/sink connectors. See [these instructions](./docker-image/CONNECT_IMAGE.md) for full steps.
### *(You may skip this step if you are able to use one of the connectors packaged inside cp-server-connect-operator)*

```
# Check/change contents of docker-image/Dockerfile for connectors you wish to add
# Build your image:
docker build -t alecpowell18/connect-custom:0.1 .
docker push alecpowell18/connect-custom:0.1
```
Publish the image to a place / repo where your k8s cluster will be able to pull it.


## 2) Deploy Operator
*(Pre-req: spin up a k8s cluster)*

Download Confluent Operator:
```
wget https://platform-ops-bin.s3-us-west-1.amazonaws.com/operator/confluent-operator-1.6.0-for-confluent-platform-6.0.0.tar.gz
tar -xvf confluent-operator-1.6.0-for-confluent-platform-6.0.0.tar.gz
cd confluent-operator/
```

Create a new values.yaml file for your deployment
```
cp helm/providers/azure.yaml values.yaml
vim values.yaml  # update license key, region and zones, enable & configure LB for external access (if desired)
```

Change # replicas, docker image name (to refer to the image you just built), for the Connect cluster in values.yaml
Also, make sure to add your API Key and API Secret for access to Confluent Cloud inside global.sasl.username / password.
```
vim values.yaml
```

Create a namespace for Confluent platform inside your k8s cluster.
```
kubectl create namespace confluent
kubectl config set-context --current --namespace confluent
```

Deploy Operator, using path to confluent-operator directory inside /helm/ directory to pick up the Helm charts.
```
helm install operator ./helm/confluent-operator --namespace confluent --values values.yaml --set operator.enabled=true
```

## 3) Deploy the Connect cluster
First: Make sure to create your API Keys and/or Service Account inside Confluent Cloud: https://docs.confluent.io/current/cloud/access-management/service-account.html

Next: Make sure that the Kafka cluster dependency is correctly linked to your Confluent Cloud cluster and API key inside values.yaml in `connect` section:
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

*Note*: Check `resources` against your k8s limits, and make sure the pod size aligns with your expectations (in prod, you might want >1 GB for heap space). See "Additional Note" section for more details & recommendations.
```
resources:
  requests:
    cpu: 200m
    memory: 256Mi
jvmConfig:
  heapSize: 256M
```

Finally, deploy the Helm chart for Connect.
```
helm install connect ./helm/confluent-operator --namespace confluent --values values.yaml --set connect.enabled=true
```

## 4) Deploy Confluent Control Center
Then, inside values.yaml, make sure that the dependency for Kafka cluster is correctly linked to your Confluent Cloud cluster and API key in `controlcenter` section:
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
  ## if deploying ksqlDB as well, set to true
  ksql:
    enabled: false
    url: http://ksql:9088
  schemaRegistry:
    enabled: false
    url: http://schemaregistry:8081
```

Now, deploy the Helm chart for Control Center.
```
helm install controlcenter ./helm/confluent-operator --namespace confluent --values values.yaml --set controlcenter.enabled=true
```

## 5) Confirm the k8s pods are spun up and ready.
```
kubectl get pods -n confluent
kubectl get services -n confluent
```

## 6) Start your connector(s)

You can either deploy through GUI at Confluent Control Center, or by exposing the REST API for Connect locally.

#### A) Deploy through Control Center

Port forward the Control Center pod's traffic locally (in this case, forwards all traffic to localhost:9021):
```
kubectl -n confluent port-forward controlcenter-0 9021:9021
```

Deploy connectors through the Connect workflow in the UI, by navigating your browser to port 9021.
```
open http://localhost:9021  # username / pw = admin / Developer1
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
*Per datagen-users-config.json, CREATE the Kafka topic first in the CCloud GUI.
Or , from the CCloud CLI:*
```
ccloud kafka topic create users
```
Start the datagen connector
```
curl -H "Content-Type: application/json" --data @connectors-config/datagen-users-config.json localhost:8083/connectors
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

## 7) Cleanup
```
helm delete controlcenter
helm delete connect
helm delete operator
```
## 8*) Running in Production
Note that the default settings in this repo will spin up Connect worker pods of only 0.2CPU and 256MB RAM. For a production deployment, you will likely want to increase the cpu > 1000m and memory >= 1G with jvmHeap >= 1G as well.


## Additional notes:

### Confluent Cloud Schema Registry
If you are using the Confluent Cloud Schema Registry with any connectors (not the local standalone SR which can be deployed using the Operator), add the following configs to the PSC file in `confluent-operator/charts/connect/templates/connect-psc.yaml`, in the "connect.properties" section. Have your SR API key & secret ready:
```
basic.auth.credentials.source=USER_INFO
basic.auth.user.info=$SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET
key.converter.basic.auth.credentials.source=USER_INFO
key.converter.schema.registry.basic.auth.user.info=$SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET
value.converter.basic.auth.credentials.source=USER_INFO
value.converter.schema.registry.basic.auth.user.info=$SCHEMA_REGISTRY_API_KEY:$SCHEMA_REGISTRY_API_SECRET
```
Don't forget to reference the SR URL in the `connect` section of `values.yaml` like so:
```
    schemaRegistry:
      enabled: true
      url: "https://psrc-abcde.us-east-2.aws.confluent.cloud"
```


### Running Enterprise-licensed connectors
Additionally, if deploying a connector which requires a Confluent license, add the extra configuration detailed in the "Connecting to Confluent Cloud" section of this KnowledgeBase doc: https://support.confluent.io/hc/en-us/articles/360040692592-How-to-include-Security-Configuration-for-License-access-into-a-Connector


### Overrides for Confluent metrics reporter
If running the Confluent metrics reporter, the Control Center pod may fail to start up. Per Confluent Cloud requirements, one configuration inside the C3 Helm chart must be updated.

Edit the PSC file in `confluent-operator/charts/controlcenter/templates/controlcenter-psc.yaml` to include the config:
```
confluent.metrics.topic.max.message.bytes=8388608
```


### Overrides required to run ksqlDB
This model can be extended to ksqlDB using the `ksql` configuration inside `values.yaml`. However, note the following configs which must be overwritten in the PSC in `confluent-operator/charts/ksql/templates/ksql-psc.yaml`:
```
ksql.internal.topic.replicas=3
ksql.streams.replication.factor=3
ksql.logging.processing.topic.replication.factor=3
```


### Overrides required to run Replicator
Update the PSC in `confluent-operator/charts/connect/templates/connect-psc.yaml`:
```
connector.client.config.override.policy=All
```
