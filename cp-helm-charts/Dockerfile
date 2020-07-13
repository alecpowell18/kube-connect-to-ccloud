FROM confluentinc/cp-server-connect-base:5.5.0

ARG CONNECTOR_VERSION=5.4.1
ENV CONNECT_PLUGIN_PATH: "/usr/share/java,/connect-plugins,/usr/share/confluent-hub-components"
#RUN confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:0.3.2
RUN confluent-hub install --no-prompt confluentinc/kafka-connect-replicator:${CONNECTOR_VERSION}
