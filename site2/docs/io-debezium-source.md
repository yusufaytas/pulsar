---
id: io-debezium-source
title: Debezium source onnector
sidebar_label: Debezium source connector
---

The Debezium source connector pulls messages from MySQL or PostgreSQL to Pulsar topics.

## Configuration 

The configuration of the Debezium source connector has the following properties.

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `task.class` | true | null | A source task class that implemented in Debezium. |
| `database.hostname` | true | null | The address of a database server. |
| `database.port` | true | null | The port number of a database server.|
| `database.user` | true | null | The name of a database user that has the required privileges. |
| `database.password` | true | null | The password for a database user that has the required privileges. |
| `database.server.id` | true | null | The connector’s identifier that must be unique within a database cluster and similar to the database’s server-id configuration property. |
| `database.server.name` | true | null | The logical name of a database server/cluster, which forms a namespace and it is used in all the names of Kafka topics to which the connector writes, the Kafka Connect schema names, and the namespaces of the corresponding Avro schema when the Avro Connector is used. |
| `database.whitelist` | false | null | A list of all databases hosted by this server which is monitored by the  connector.<br/><br/> This is optional, and there are other properties for listing databases and tables to include or exclude from monitoring. |
| `key.converter` | true | null | The converter provided by Kafka Connect to convert record key. |
| `value.converter` | true | null | The converter provided by Kafka Connect to convert record value.  |
| `database.history` | true | null | The name of the database history class. |
| `database.history.pulsar.topic` | true | null | The name of the database history topic where the connector writes and recovers DDL statements. <br/><br/>**Note: this topic is for internal use only and should not be used by consumers.** |
| `database.history.pulsar.service.url` | true | null | Pulsar cluster service URL for history topic. |
| `pulsar.service.url` | true | null | Pulsar cluster service URL. |
| `offset.storage.topic` | true | null | Record the last committed offsets that the connector successfully completes. |

## Example of MySQL

You need to create a configuration file before using the Pulsar Debezium connector.

### Configuration 

You can use one of the following methods to create a configuration file.

* JSON 

    ```json
    {
        "database.hostname": "localhost",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "184054",
        "database.server.name": "dbserver1",
        "database.whitelist": "inventory",
        "database.history": "org.apache.pulsar.io.debezium.PulsarDatabaseHistory",
        "database.history.pulsar.topic": "history-topic",
        "database.history.pulsar.service.url": "pulsar://127.0.0.1:6650",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "pulsar.service.url": "pulsar://127.0.0.1:6650",
        "offset.storage.topic": "offset-topic"
    }
    ```

* YAML 

    You can create a `debezium-mysql-source-config.yaml` file and copy the [contents](https://github.com/apache/pulsar/blob/master/pulsar-io/debezium/mysql/src/main/resources/debezium-mysql-source-config.yaml) below to the `debezium-mysql-source-config.yaml` file.

    ```yaml
    tenant: "public"
    namespace: "default"
    name: "debezium-mysql-source"
    topicName: "debezium-mysql-topic"
    archive: "connectors/pulsar-io-debezium-mysql-{{pulsar:version}}.nar"
    parallelism: 1

    configs:

        ## config for mysql, docker image: debezium/example-mysql:0.8
        database.hostname: "localhost"
        database.port: "3306"
        database.user: "debezium"
        database.password: "dbz"
        database.server.id: "184054"
        database.server.name: "dbserver1"
        database.whitelist: "inventory"
        database.history: "org.apache.pulsar.io.debezium.PulsarDatabaseHistory"
        database.history.pulsar.topic: "history-topic"
        database.history.pulsar.service.url: "pulsar://127.0.0.1:6650"

        ## KEY_CONVERTER_CLASS_CONFIG, VALUE_CONVERTER_CLASS_CONFIG
        key.converter: "org.apache.kafka.connect.json.JsonConverter"
        value.converter: "org.apache.kafka.connect.json.JsonConverter"

        ## PULSAR_SERVICE_URL_CONFIG
        pulsar.service.url: "pulsar://127.0.0.1:6650"

        ## OFFSET_STORAGE_TOPIC_CONFIG
        offset.storage.topic: "offset-topic"
    ```

### Usage

This example shows how to change the data of a MySQL table using the Pulsar Debezium connector.

1. Start a MySQL server with a database from which Debezium can capture changes.

    ```bash
    $ docker run -it --rm \
    --name mysql \
    -p 3306:3306 \
    -e MYSQL_ROOT_PASSWORD=debezium \
    -e MYSQL_USER=mysqluser \
    -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:0.8
    ```

2. Start a Pulsar service locally in standalone mode.

    ```bash
    $ bin/pulsar standalone
    ```

3. Start the Pulsar Debezium connector in local run mode using one of the following methods.

     * Use the **JSON** configuration file as shown previously. 
   
        Make sure the nar file is available at `connectors/pulsar-io-debezium-mysql-{{pulsar:version}}.nar`.

        ```bash
        $ bin/pulsar-admin source localrun \
        --archive connectors/pulsar-io-debezium-mysql-{{pulsar:version}}.nar \
        --name debezium-mysql-source --destination-topic-name debezium-mysql-topic \
        --tenant public \
        --namespace default \
        --source-config '{"database.hostname": "localhost","database.port": "3306","database.user": "debezium","database.password": "dbz","database.server.id": "184054","database.server.name": "dbserver1","database.whitelist": "inventory","database.history": "org.apache.pulsar.io.debezium.PulsarDatabaseHistory","database.history.pulsar.topic": "history-topic","database.history.pulsar.service.url": "pulsar://127.0.0.1:6650","key.converter": "org.apache.kafka.connect.json.JsonConverter","value.converter": "org.apache.kafka.connect.json.JsonConverter","pulsar.service.url": "pulsar://127.0.0.1:6650","offset.storage.topic": "offset-topic"}'
        ```

    * Use the **YAML** configuration file as shown previously.
  
        ```bash
        $ bin/pulsar-admin source localrun \
        --source-config-file debezium-mysql-source-config.yaml
        ```

4. Subscribe the topic _sub-products_ for the table _inventory.products_.

    ```bash
    $ bin/pulsar-client consume -s "sub-products" public/default/dbserver1.inventory.products -n 0
    ```

5. Start a MySQL client in docker.

    ```bash
    $ docker run -it --rm \
    --name mysqlterm \
    --link mysql \
    --rm mysql:5.7 sh \
    -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
    ```

6. A MySQL client pops out. 
   
   Use the following commands to change the data of the table _products_.

    ```
    mysql> use inventory;
    mysql> show tables;
    mysql> SELECT * FROM  products;
    mysql> UPDATE products SET name='1111111111' WHERE id=101;
    mysql> UPDATE products SET name='1111111111' WHERE id=107;
    ```

    In the terminal window of subscribing topic, you can find the data changes have been kept in the _sub-products_ topic.

## Example of PostgreSQL

You need to create a configuration file before using the Pulsar Debezium connector.

### Configuration

You can use one of the following methods to create a configuration file.

* JSON 

    ```json
    {
        "database.hostname": "localhost",
        "database.port": "5432",
        "database.user": "postgres",
        "database.password": "postgres",
        "database.dbname": "postgres",
        "database.server.name": "dbserver1",
        "schema.whitelist": "inventory",
        "pulsar.service.url": "pulsar://127.0.0.1:6650"
    }
    ```

* YAML 

    You can create a `debezium-postgres-source-config.yaml` file and copy the [contents](https://github.com/apache/pulsar/blob/master/pulsar-io/debezium/postgres/src/main/resources/debezium-postgres-source-config.yaml) below to the `debezium-postgres-source-config.yaml` file.

    ```yaml
    tenant: "public"
    namespace: "default"
    name: "debezium-postgres-source"
    topicName: "debezium-postgres-topic"
    archive: "connectors/pulsar-io-debezium-postgres-{{pulsar:version}}.nar"
    parallelism: 1

    configs:

        ## config for pg, docker image: debezium/example-postgress:0.8
        database.hostname: "localhost"
        database.port: "5432"
        database.user: "postgres"
        database.password: "postgres"
        database.dbname: "postgres"
        database.server.name: "dbserver1"
        schema.whitelist: "inventory"

        ## PULSAR_SERVICE_URL_CONFIG
        pulsar.service.url: "pulsar://127.0.0.1:6650"
    ```

### Usage

This example shows how to change the data of a PostgreSQL table using the Pulsar Debezium connector.


1. Start a PostgreSQL server with a database from which Debezium can capture changes.

    ```bash
    $ docker pull debezium/example-postgres:0.8
    $ docker run -d -it --rm --name pulsar-postgresql -p 5432:5432  debezium/example-postgres:0.8
    ```

2. Start a Pulsar service locally in standalone mode.

    ```bash
    $ bin/pulsar standalone
    ```

3. Start the Pulsar Debezium connector in local run mode using one of the following methods.

   * Use the **JSON** configuration file as shown previously. 
     
     Make sure the nar file is available at `connectors/pulsar-io-debezium-postgres-{{pulsar:version}}.nar`.

        ```bash
        $ bin/pulsar-admin source localrun \
        --archive connectors/pulsar-io-debezium-postgres-{{pulsar:version}}.nar \
        --name debezium-postgres-source \
        --destination-topic-name debezium-postgres-topic \
        --tenant public \
        --namespace default \
        --source-config '{"database.hostname": "localhost","database.port": "5432","database.user": "postgres","database.password": "postgres","database.dbname": "postgres","database.server.name": "dbserver1","schema.whitelist": "inventory","pulsar.service.url": "pulsar://127.0.0.1:6650"}'
        ```
   
   * Use the **YAML** configuration file as shown previously.
      
        ```bash
        $ bin/pulsar-admin source localrun  \
        --source-config-file debezium-postgres-source-config.yaml
        ```

4. Subscribe the topic _sub-products_ for the _inventory.products_ table.

    ```
    $ bin/pulsar-client consume -s "sub-products" public/default/dbserver1.inventory.products -n 0
    ```

5. Start a PostgreSQL client in docker.
   
    ```bash
    $ docker exec -it pulsar-postgresql /bin/bash
    ```

6. A MySQL client pops out. 
   
   Use the following commands to change the data of the table _products_.

    ```
    psql -U postgres postgres
    postgres=# \c postgres;
    You are now connected to database "postgres" as user "postgres".
    postgres=# SET search_path TO inventory;
    SET
    postgres=# select * from products;
     id  |        name        |                       description                       | weight
    -----+--------------------+---------------------------------------------------------+--------
     102 | car battery        | 12V car battery                                         |    8.1
     103 | 12-pack drill bits | 12-pack of drill bits with sizes ranging from #40 to #3 |    0.8
     104 | hammer             | 12oz carpenter's hammer                                 |   0.75
     105 | hammer             | 14oz carpenter's hammer                                 |  0.875
     106 | hammer             | 16oz carpenter's hammer                                 |      1
     107 | rocks              | box of assorted rocks                                   |    5.3
     108 | jacket             | water resistent black wind breaker                      |    0.1
     109 | spare tire         | 24 inch spare tire                                      |   22.2
     101 | 1111111111         | Small 2-wheel scooter                                   |   3.14
    (9 rows)
    
    postgres=# UPDATE products SET name='1111111111' WHERE id=107;
    UPDATE 1
    ```

    In the terminal window of subscribing topic, you can receive the following messages.
        
    ```bash
    ----- got message -----
    {"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.products.Key"},"payload":{"id":107}}�{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"name"},{"type":"string","optional":true,"field":"description"},{"type":"double","optional":true,"field":"weight"}],"optional":true,"name":"dbserver1.inventory.products.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"name"},{"type":"string","optional":true,"field":"description"},{"type":"double","optional":true,"field":"weight"}],"optional":true,"name":"dbserver1.inventory.products.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":true,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"string","optional":false,"field":"db"},{"type":"int64","optional":true,"field":"ts_usec"},{"type":"int64","optional":true,"field":"txId"},{"type":"int64","optional":true,"field":"lsn"},{"type":"string","optional":true,"field":"schema"},{"type":"string","optional":true,"field":"table"},{"type":"boolean","optional":true,"default":false,"field":"snapshot"},{"type":"boolean","optional":true,"field":"last_snapshot_record"}],"optional":false,"name":"io.debezium.connector.postgresql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.products.Envelope"},"payload":{"before":{"id":107,"name":"rocks","description":"box of assorted rocks","weight":5.3},"after":{"id":107,"name":"1111111111","description":"box of assorted rocks","weight":5.3},"source":{"version":"0.9.2.Final","connector":"postgresql","name":"dbserver1","db":"postgres","ts_usec":1559208957661080,"txId":577,"lsn":23862872,"schema":"inventory","table":"products","snapshot":false,"last_snapshot_record":null},"op":"u","ts_ms":1559208957692}}
    ```