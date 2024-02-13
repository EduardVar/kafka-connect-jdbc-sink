# Db2 Scarf&trade;

`Last Updated February 13, 2024`

This documentation gives the steps to setup, connect to, and utilize Kafka Sink Connector with JDBC connectors.

## Before You Begin

### Installation
1. Clone the [kafka-connect-jdbc-sink](https://github.com/ibm-messaging/kafka-connect-jdbc-sink/tree/main) repository with the following command:
    ```
    git clone git@github.com:ibm-messaging/kafka-connect-jdbc-sink.git
    ```
    > Note: if you don't have access to this repository, fork it and then clone it.
2. Make sure [Apache Kafka installed](https://kafka.apache.org/downloads) is installed. (Ex: Version 3.6.0) Make note of the port you will be using (default: `9092`). Save the location of the Kafka installation to an environment variable:
    ```
    KAFKA_BROKER_PORT=9092
    KAFKA_INSTALL_DIR=<PATH-TO-KAFKA-INSTALLATION>
    ``` 
3. [Zookeper](https://kafka.apache.org/quickstart#quickstart_startserver) comes with the Kafka installation. Make note of the port you will be using (default: `2181`).

> **NOTE**: It's assumed you have a Db2 installation that's TCP IP enabled.

### Setup
1. Enter the cloned repository.
    ```
    cd kafka-connect-jdbc-sink
    ```
2. Create a directory to store `.jar` files that will be used with the built kafka connector. Ex:
    ```
    mkdir kafka_jar_files   # Just an exmaple
    KAFKA_JAR_FILES=$(cd kafka_jar_files; pwd; cd ..)   # This saves the absolute path of your created folder
    ```
3. Build the connector using [maven](https://maven.apache.org/):
    ```
    mvn clean package
    ```
    This generates a `.jar` file in the `target/` directory of the cloned repository, which now you can put in the `KAFKA_JAR_FILES` path.
    ```
    cp target/kafka-connect-jdbc-sink-1.0.0-SNAPSHOT-jar-with-dependencies.jar $KAFKA_JAR_FILES/
    ```
4. Download the [dc2jcc.jar](https://www.ibm.com/support/pages/db2-jdbc-driver-versions-and-downloads) and move it into your `KAFKA_JAR_FILES` directory.
5. Copy the entire `config` directory from the repository to your `KAFKA_INSTALL_DIR`:
    ```
    cp -r config $KAFKA_INSTALL_DIR/
    ```
5. Make sure your environment variables are updated to be pointing to `KAFKA_JAR_FILES`. Ex:
    ```
    export CLASSPATH="$CLASSPATH:$KAFKA_JAR_FILES"
    ```

## Configuration

1. In your Kafka installation (`KAFKA_INSTALL_DIR`), set up a topic. Below is an example of paramters used for configuring Kafka to your Db2 database.
    ```
    TOPIC_NAME="kafka_test"
    PARTITIONS=3
    REPLICATION=1
    ZOOKEEPER_PORT=2181
    ```
    Now, plug everything in to create a Kafka topic:
    ```
    kafka-topics --create --topic $TOPIC_NAME --partitions $PARTITIONS --replication-factor $REPLICATION --zookeeper 127.0.0.1:$ZOOKEEPER_PORT
    ```
2. In your `KAFKA_INSTALL_DIR`, in the `config` folder, you'll need to create a file called `jdbc-sink.properties`:
    ```
    cd $KAFKA_INSTALL_DIR
    vi config/jdbc-sink.properties
    ```
3. You will need to fill `jdbc-sink.properties` out with the information in the values surrounded by `< >` with the values of your ***JDBC database***. Template:
    ```
    # A simple example that copies from a topic to a JDBC database.
    # The first few settings are required for all connectors:
    # a name, the connector class to run, and the maximum number of tasks to create:
    name=<CONNECTOR-NAME>               # Example: jdbc-sink-connector
    connector.class=<CLASS-PATH>        # Example: com.ibm.eventstreams.connect.jdbcsink.JDBCSinkConnector
    tasks.max=<TASKS_MAX>               # Example: 1    (Change as needed)

    # Below is the db2 driver (used instead of postgres)
    driver.class=com.ibm.db2.jcc.DB2Driver

    # The topics to consume from - required for sink connectors
    topics=<TOPIC-NAME>    # Example: kafka_test from $TOPIC_NAME

    # Configuration specific to the JDBC sink connector.
    # We want to connect to a SQLite database stored
    # in the file test.db and auto-create tables.
    connection.url=<CONNECTION_URL_OF_YOUR_DATABASE>    # Example: jdbc:db2://localhost:60006/BLUDB
    connection.user=<CONNECTION_USER>
    connection.password=<CONNECTION_PASSWORD>
    connection.ds.pool.size=<POOL_SIZE>                 # Example: 5    (Update as needed)
    connection.table=<TABLE_NAME>
    insert.mode.databaselevel=true
    insert.function.value=memory_table
    put.mode=insert
    table.name.format=<[shema].[table]>                 # Example: EDU.TAB1
    auto.create=true

    # Define when identifiers should be quoted in DDL and DML statements.
    # The default is 'always' to maintain backward compatibility with prior versions.
    # Set this to 'never' to avoid quoting fully-qualified or simple table and column names.
    #quote.sql.identifiers=always
    ```
Now you're set to run the Db2 Scarf&trade; connector in Standalone or Distributed Mode.

## Running your Db2 Scarf&trade; Connector
### ‼️ Before Running your Connector ‼️
Before running your connector, you need to start the Kafka Zookeeper and Kafka Broker:
```
cd $KAFKA_INSTALL_DIR
bin/zookeeper-server-start.sh config/zookeeper.properties
bin/kafka-server-start.sh config/server.properties
```
> Please make sure both the kafka and zookeper servers are running before starting ingesting.

### Running in Standalone Mode
1. Navigate to your `KAFKA_INSTALL_DIR`,
    ```
    cd $KAFKA_INSTALL_DIR
    ```
2. Edit your `config/connect-standalone.properties` file and set these two settings to `true`:
    ```
    key.converter.schemas.enable=true
    value.converter.schemas.enable=true
    ```
    Additionally in the file, make sure `plugin.path` is set to the the location of `KAFKA_JAR_FILES`:
    ```
    plugin.path=<PATH-OF-KAFKA_JAR_FILES>
    ```
3. Finally run the below command:
    ```
    bin/connect-standalone.sh config/connect-standalone.properties config/jdbc-sink.properties
    ```
Next, go the [Validation](##Validation) section or use your own producer to begin ingesting data!

### Running in Distributed Mode
In order to run the connector in distributed mode you must first register the connector with [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#what-is-kafka-connect) service by creating a JSON file in `KAFKA_INSTALL_DIR/config/jdbc-connector.json` with the format below:
```
{
  "name": "<CONNECTOR-NAME>",
  "config": {
    "connector.class": "<CLASS-PATH>",
    "tasks.max": "<TASKS_MAX>",
    "topics": "<TOPIC-NAME>",
    "connection.url": "<CONNECTION_URL_OF_YOUR_DATABASE>",
    "connection.user": "<CONNECTION_USER>",
    "connection.password": "<CONNECTION_PASSWORD>",
    "connection.ds.pool.size": <POOL_SIZE>,
    "insert.mode.databaselevel": true,
    "table.name.format": "<[shema].[table]>"
  }
}
```
Once you fill in the `jdbc-connector.json` file, navigate back to your `KAFKA_INSTALL_DIR` and register the connector.
```
cd $KAFKA_INSTALL_DIR
connect-distributed connect-distributed.properties
curl -s -X POST -H 'Content-Type: application/json' --data @config/jdbc-connector.json http://localhost:8083/connectors
```
You can verify that your connector was properly registered by going to http://localhost:8083/connectors which should return a full list of available connectors. This JSON connector profile will be propegated to all workers across the distributed system. After following these steps your connector will now run in distributed mode.


## Validation
This section will be to test that your configured Kafka connector can injest data.

1. Start a kafka producer:
    ```
    kafka-console-producer --broker-list localhost:$KAFKA_BROKER_PORT --topic $TOPIC_NAME
    ```
2. Create a schema JSON record and enter it into the producer terminal from step 1. Example record:
    ```
    {"schema": {"type": "struct","fields": [{"type": "string","optional": false,"field": "Name"}, {"type": "string","optional": false,"field": "company"}],"optional": false,"name": "Person"},"payload": {"Name": "Roy Jones","company": "General Motors"}}
    ```
    > Pasting this record with no table in the ***JDBC database*** creates a new `table` with the `schema`. This JSON record assumes a table has four columns, so if you already have an existing `table` make sure the table with that `schema` has four columns. 
3. Open up the command-line client of your ***JDBC database*** and verify that a record has been added into the target database table. If the database table did not exist prior to this, it would have been created by this process.
    > Be sure to target the proper database by using `\c <database_name>` or `USE <database_name>;`.
4. You should be able to see your newly created record added to your ***JDBC database*** table. 
    
    Example from schema in step 2, selecting from it gives:
    ```
    select * from company;

     id |   timestamp   |   name    |    company     
     ----+------+---------------+-----------+----------------
     1 | 1587969949600 | Roy Jones |  General Motors
    ```

### Using the Dashboard
[LOREM IPSUM]

## Differences between older repo/new repo
- Contains the functionality for the `MEMORY_TABLE` mode described in the documentation
- TODO: Future performance testing for Distributed mode (currently we're only testing standalone mode)

## Extra Links / Related Topics
- [kafka-connect-jdbc-sink Repository with ReadMe](https://github.com/ibm-messaging/kafka-connect-jdbc-sink/tree/main)
- Limitations: Needs to be restarted every single record set ingest.
