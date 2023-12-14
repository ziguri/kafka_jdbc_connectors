# Setup KAFKA Connect JDBC connectors

This tutorial aims to help to setup the KAFKA local environment.
It consists in configuring 1 Source connector and 1 Sink connector for the table T_FACTORY.

## The Factory POC

Along this setup you will be able to build a complete setup of local Kafka running in Docker along with a local database for testing purpose. The goal is to integrate this database table ```T_FACTORY``` with Kafka in order to see JDBC Kafka Source and Sink connectors in action.

It supports the automatic streaming of messages, stored in the T_FACTORY table, into 3 different topics ```assembly_topic``` , ```component_topic``` and ```part_topic````. The result of this integration is set by a stream back from Kafka to de T_FACTORY table, managed by the JDBC Connector Sink, with the information if the message was streamed successfully for the expected topic.


## Requirements

1 - Make sure you have docker already installed. If not please install the docker desktop version available on the official
site [here](https://www.docker.com/products/docker-desktop) (That will install also docker-compose which we will need later).

## Starting the environment

Open your terminal and run (assuming you start from the root of the project):

```bash
cd kafka
docker-compose -f docker-compose.yml up -d
```
## Create the database table 
Just add the ```T_FACTORY``` using any DB client.
```sql
CREATE TABLE T_FACTORY
(
    id           SERIAL PRIMARY KEY,
    version      INT          NULL,
    kafka_status VARCHAR(25)  NOT NULL,
    factory_name VARCHAR(255) NULL,
    factory_info    TEXT     NOT NULL,
    kafka_topic  VARCHAR(100) NOT NULL,
    creator      VARCHAR(255) NULL,
    created      DATE          NULL,
    modifier     VARCHAR(255) NULL,
    modified     DATE
);

```

## Creating the topics

Just enter in the zookeeper container:
```bash
docker exec -it zookeeper bash
# or the one above if you are using mintty
winpty docker exec -it zookeeper bash
```
or just go to docker desktop, mouse-over the 'zookeeper' container and click 'CLI'.

Once you are inside container:
```bash
kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic component_topic &&
kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic part_topic &&
kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic assembly_topic
```
(Optional) To delete just:
```bash
kafka-topics --zookeeper localhost:2181 --delete --topic component_topic &&
kafka-topics --zookeeper localhost:2181 --delete --topic part_topic &&
kafka-topics --zookeeper localhost:2181 --delete --topic assembly_topic
```

## Creating the connectors
### Creating the source connector
The local environment just uses one source connector, responsible for sending the information from staging table to the kafka topics.

To create the source connector just:
```bash
cd kafka/connectors
curl -X PUT -H "Content-Type: application/json" -d @create_jdbc_source.json http://localhost:8083/connectors/source-jdbc-postgresql/config -o /dev/null -s -w "%{http_code}\n"
```
Response should be 201.

Note: the host here (localhost:8083 means we are using the kafka-connect container, if kafka-connect by some reason is not using this port just change it on the url above)

### Creating the sink connector

The sink connector has knowledge about all the topics and has the responsibility to return the feedback to the factory table that the message has been delivered.

That success feedback is marked as 'STREAMED' in the column kafka_status of that record in T_FACTORY.

Just run:
```bash
cd kafka/connectors
curl -X PUT -H "Content-Type: application/json" -d @create_jdbc_sink_all_topics.json http://localhost:8083/connectors/sink-jdbc-postgresql-all-topics/config -o /dev/null -s -w "%{http_code}\n"
```
Response should be 201.

## Creating data to be consumed by the connectors
Well, simply provide data to the T_FACTORY as so:
```sql
insert into T_FACTORY (version, kafka_status, factory_name, factory_info, kafka_topic, creator, created,
                       modifier, modified)
values (1, 'NEW', 'FactoryAB', '{"assembly_line": "123XPT2"}', 'assembly_topic', 'olneves', now(), '', null);
```
Notice that you decide the topic where this message goes in the column kafka_topic. In this example it's going to the assembly_topic. You might want to test also sending messages to different topics as part_topic or component_topic.

## How to access information
After the environment is up alongside the topics and connectors setup, it's time to navigate through the information available.

### Connect to the ksqlDB (the kafka internal database)
In order to query lots of information about the topics and the connectors we can use this kafka internal database.

Option 1: 
In your bash shell just run:
```bash
docker exec -it ksqldb ksql
# or the one above if you are using mintty
winpty docker exec -it ksqldb ksql
```

Option 2:
Just go to docker desktop, mouse-over the 'ksqldb' container and click 'CLI'. Then write 'ksql' and you will be redirected to the ksql prompt.

To exit the ksql prompt just:
```bash
ksql> exit
```

### Listing connectors
In ksqlDB, just run:
```bash
ksql> show connectors;
```
### Listing topics
In ksqlDB, just run:
```bash
ksql> show topics;
```
### Show messages from a specific topic
In ksqlDB, just run:
```bash
ksql> PRINT assembly_topic;
ksql> PRINT assembly_topic FROM BEGINNING;
```

### Deleting connectors
In this example below you delete the source and sink connector created above.
```bash
cd kafka/connectors
curl -X DELETE http://localhost:8083/connectors/source-jdbc-postgresql -o /dev/null -s -w "%{http_code}\n"
curl -X DELETE http://localhost:8083/connectors/sink-jdbc-postgresql-all-topics -o /dev/null -s -w "%{http_code}\n"
```
Response should be 204.

### Checking the status of connectors
#### For the Source Connector
```bash
cd kafka/connectors
curl -X GET http://localhost:8083/connectors/source-jdbc-postgresql/tasks/0/status
```
#### For the Sink Connector
```bash
cd kafka/connectors
curl -X GET http://localhost:8083/connectors/sink-jdbc-postgresql-all-topics/tasks/0/status
```

This command is useful to find ERRORs in the connector creation. A stack trace will appear in the response body if any.

### Checking the status of the messages sent from the Factory
The messages to be sent are always on the T_FACTORY table.
When creating the message to be sent (as a new entry in T_FACTORY) the record has the status 'NEW'.

When the record is successfully sent to the topic (with the help of the connectors) the status of that record changes to 'STREAMED'.

We can see this simply by querying the table and checking the status of our record.
```sql
select * from T_FACTORY;
```
