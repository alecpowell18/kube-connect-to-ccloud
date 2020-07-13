Starting Confluent REST Proxy in k8s against Confluent Cloud.

```
minikube start --cpus=4 --memory=4G
kubectl create secret generic kafka-rest-config --from-file=kafka-rest.properties
kubectl apply -f kafka-rest-proxy-deployment.yaml
kubectl port-forward service/kafka-cloud-proxy 8082:8082
//create topic in ccloud
curl http://localhost:8082/topics
//produce to topic now
curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" \
      -H "Accept: application/vnd.kafka.v2+json" \
      --data '{"records":[{"value":{"foo":"bar"}}]}' "http://localhost:8082/topics/rest-test"
//follow quickstart at https://docs.confluent.io/current/kafka-rest/quickstart.html
```
