# Creating custom Docker image to include additional connectors

How to create a Docker image for custom connectors?

This guide describes how to add custom connectors into a Kubernetes Confluent Operator deployment. Confluent Operator automatically packages and deploys the following connectors when it deploys Confluent Connect and Connect workers:

1. kafka-connect-activemq
2. kafka-connect-elasticsearch
3. kafka-connect-hdfs
4. kafka-connect-IBMmq
5. kafka-connect-JDBC
6. kafka-connect-jms
7. kafka-connect-s3

The above can be used out of the box in a Confluent Operator deployment in Kubernetes. However, if you’d like to use the vast array of connectors in Confluent’s Connector Hub or an Open Source Connector, a custom layered container needs to be built from Confluent Operator’s Connect Docker image. The private docker registry is required to work as the custom docker image need to be pushed to private docker registry.

## Step by Step Guide

#### Find your connector that needs to be deployed into a Confluent Operator Kubernetes cluster

[Confluent Hub](https://www.confluent.io/hub)
(You will be downloading connectors using the `confluent-hub` command)

#### Create a Dockerfile. 

The docker file should use the Confluent Operator connect image as the base image.

Create Dockerfile with the following information for connector confluentinc-kafka-connect-datagen:

```
FROM confluentinc/cp-server-connect-operator:6.0.0.0
USER root
RUN confluent-hub install confluentinc/kafka-connect-datagen:0.4.0 --no-prompt
```

#### Build Docker image

Build the image. Push the container to the private docker repository you may be using.

```
docker build -t alecpowell18/connect-custom:0.1 .
docker push alecpowell18/connect-custom:0.1
```

#### Refer to new image in `values.yaml`

Use this new connect image by updating `providers/values.yaml` file as follows:
```
connect:...image:
  repository: <your-repo>/connect-custom
  tag: 1.0
```

#### Go! 

Deploy Connect using Confluent Operator HELM charts.
