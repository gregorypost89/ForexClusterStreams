# Currency Pairs

## This program uses Kafka with Zookeeper to take in currency pair data as a producer and ship to consumers

### Step 1 - Start the Kafka Server

We will use the Zookeeper convenience script that comes with Kafka.  We'll start by creating the single node instance.

```
bin/zookeeper-server-start.sh config/zookeeper.properties
```

Next we'll expand our cluster to three nodes.  We do this by making a config file for each broker

```
cp config/server.properties config/server-1.properties
cp config/server.properties config/server-2.properties
```

When we created our single node instance, it will be running on **localhost:9092**.
Our next two nodes cannot conflict with each other as we are running these on the same machine, so we need to edit the configuration of listeners to reflect this:

```
config/server-1.properties:
    broker.id=**1**
    listeners=**PLAINTEXT://:9093**
    log.dirs=/tmp/kafka-logs-1

config/server-2.properties:
    broker.id=**2**
    listeners=**PLAINTEXT://:9094**
    log.dirs=/tmp/kafka-logs-2
```

Next, we start the Kafka server with our three servers.

```
bin/kafka-server-start.sh config/server.properties &
...
bin/kafka-server-start.sh config/server-1.properties &
...
bin/kafka-server-start.sh config/server-2.properties &
```

### Step 2 - Creating our topic

We will name our topic "pairs".  
It is good practice to have 3 replicas and 5 partitions to tolerate failure.

```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 3 --partitions 5 --topic pairs
```

Test this by running the output script and we should see our topic return as a result

```
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
pairs
```

### Step 3 - Creating our connectors with Kafka Connect

We will start two connectors in standalone mode

To do this, we provide three configuration files as parameters:

    - connect-standalone.sh : configuration for Kafka Connect
    - connect-file-source.properties: source connector that reads lines from our input file and publishes them to the **pairs** topic
    - connect-file-sink.properties: sink connector that reads messages from the **pairs** topic and produces each line into an output file.

We need to update our sink connector properties to listen to the **pairs** topic.
We can also optionally edit the desired destination file, which is test.sink.txt as default.

```
name=local-file-sink
connector.class=FileStreamSink
tasks.max=1
file=/results.txt
topics=pairs
```

Now we need to edit our file source connector properties to reflect the file we are listening to.
The source data is stored in **rates.json** in this repository
If using an API that gets this data and stores to a directory, we can specify the directory where this JSON would be contained.

```
name=local-file-sink
connector.class=FileStreamSink
tasks.max=1
file=/rates.json
topics=pairs
```

To parse the json data, we can use **jq** which is a command line JSON processor to take our JSON input and process it in a meaningful way to be published to and consumed from our topic. 

```
jq sampledata.json | ./kafka-console-producer --bootstrap-server localhost:9092 --topic pairs --zookeeper localhost:2181
```

(Remember to configure for your server instance.  For example, if running on Hortonworks Sandbox, configure localhost:9092 to the appropriate parameters, for example sandbox.hortonworks.com:6667)

If the data is parsing incorrectly due to the JSON format, for example each dictionary is being passed individually, we can then pass a **resource configuration** (-rc) and utilize a resource configuration file that will determine how our producer processes this data

```
jq -rc . sampledata.json | ./kafka-console-producer --bootstrap-server localhost:9092 --topic pairs --zookeeper localhost:2181
```
