FROM confluentinc/cp-server-connect-operator:6.0.0.0
USER root 
RUN confluent-hub install confluentinc/kafka-connect-datagen:0.4.0 --no-prompt
RUN confluent-hub install confluentinc/kafka-connect-sftp:2.1.5 --no-prompt
RUN confluent-hub install debezium/debezium-connector-sqlserver:1.3.1 --no-prompt
RUN confluent-hub install snowflakeinc/snowflake-kafka-connector:1.5.1 --no-prompt
RUN confluent-hub install confluentinc/kafka-connect-salesforce:1.2.0 --no-prompt
