# Forex Cluster Streams

## This program uses Kafka with Zookeeper to take in currency pair data as a producer and ship to consumers

### Introduction

One of the major challenges of handling data in the financial world is the live nature of the market.  As traders of all markets are making decisions based on real time data, the processing that is performed on a cluster may necessitate this data to be retrieved at set intervals, whether daily, hourly, or even by the second if required.

The users of an app that deal with live data also need to visualize this data in real time.  Yahoo Finance is a great example of this, where the ticker shows a stock price that increases or decreases in real time accompanied by a respective green or red highlight to indicate price movement.

![TSLA Stock Price](img/tsla.gif)

But how does this happen?  How does Yahoo manage to get these real time quotes and push it so quickly to their interface, so that the user can see it insantaneously?

This is where **streaming** comes in; the steady, high-speed, and continuous transfer of data.   
Tim Berglund of Confluent statest that its best to think of streaming as an "unbounded, continuous real-time flow of records"¹,and this is a good approach to visualizing how streaming data can be useful in this context.

The reason we use Kafka is becuase no micro-batching is involved. 
These records don't just get stuffed in some directory or data-store somewhere and then pulled. 
It supports up to *millisecond* latency on per-record stream processing
Although we likely do not need this level of precision, many applications that deal with live financial data will require and depend upon this standard.

Kafka is what is known as a **publish-subscribe** messaging system.

In the real world, a publisher takes in information from around the world and publishes it to a newspaper or book.
Kafka publishers act similarly.  
The publisher is listening for specific data, like the stock price data we have above, and will publish it to a **topic** in Kafka.

Think of how the newspaper has many different sections, like Sports, Finance, Politics, Food.  These are basically different topics, and there is a publisher associated with each one of these topics.  
In Kafka, we can have the same structure.  We can set up multiple different publishers, all listening to their source of information, and publishing those to our Kafka Cluster.

The intention of this repository is to take foreign exchange data from an API, where our API calls act as our stream.  
This data which is consumed by our producer(s) into respective topics on our Kafka Cluster.

In addition, there are plans to implement **stream processors** that can take these input topics and transform them into its own output topics.  A subscriber has the ability to subscribe to a topic that contains hourly price data on a currency pair, but if they were to require some calculation on this data, such as a moving average, this would require some form of stream processor to take this data and transform it in real time for the subscriber to access.

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
    broker.id=1
    listeners=PLAINTEXT://:9093
    log.dirs=/tmp/kafka-logs-1

config/server-2.properties:
    broker.id=2
    listeners=PLAINTEXT://:9094
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
