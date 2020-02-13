# Forex Cluster Streams

## This program uses Kafka with Zookeeper to take in currency pair data as a producer and ship to consumers, as well as providing indicators on live data as trading tools for currency speculators.

## Index

#### Introduction : Summary of Kafka
#### Part 1: Establishing the Kafka Cluster


### Introduction: Summary of Kafka

One of the major challenges of handling data in the financial world is the live nature of the market.  As traders of all markets are making decisions based on real time data, the processing that is performed on a cluster may necessitate this data to be retrieved at set intervals, whether daily, hourly, or even by the second if required.

The users of an app that deal with live data also need to visualize this data in real time.  Yahoo Finance is a great example of this, where the ticker shows a stock price that increases or decreases in real time accompanied by a respective green or red highlight to indicate price movement.

![TSLA Stock Price](gif/tsla.gif)

But how does this happen?  How does Yahoo manage to get these real time quotes and push it so quickly to their interface, so that the user can see it insantaneously?

This is where **streaming** comes in: the steady, high-speed, and continuous transfer of data.   
Tim Berglund of Confluent states that its best to think of streaming as an 
"unbounded, continuous real-time flow of records"¹, and this is a good visual representation of what we are viewing in real time.


This application will transform the data as it comes in.
Our architecture of choice will be Kafka.
What does this structure look like?

Kafka is what is known as a **publish-subscribe** messaging system.

In this system, we have one or more servers running Kakfa.
These servers are known as Kafka brokers, and constitute our Kafka Cluster.

The publishers are responsible for pushing messages into our Kafka Cluster.  As an example, look at the stock trade gif above.  Imagine that as our producer, where every time the price gets updated, it pushes that input as a message into our Kafka Cluster.

On the other end, our subscribers consume the data as its produced.  Think of how the Yahoo Website is actually retrieving that information in the first place to display to our monitor.  It has an app that is subscribed to that data and utilizes it to display the resulting stock prices.

![Imgur](gif/kafkaSimpleLayout.gif)

Now when we are pulling data from an API, we are usually pulling lots of different kinds of data.  We don't want all of this data to be pushed into our cluster, because our subscriber is only looking for very specific information from all of that data.  Also, how do we separate the data we are receiving into our cluster?  We need some way to organize the flow of information into our cluster

This is where **topics** come in.  A kafka publisher can publish information into topics in our Kafka cluster to segment the information.  The subscribers then subscribe to the topics of their choice to get only the information they require. 

Lets provide a scenario to make this easier to follow.
Lets say we want to build a sports betting app.
Suppose we have a game between the Philadelphia Eagles and Dallas Cowboys which we want to monitor

In sports betting, we have a money line that indicates a wager amount and winning amount for each team.
The favored team to win this match, lets say Dallas in this case, has a money line of -120
This means someone must wager $120 to win $100.
The underdog team, which is Philadelphia, has a money line of +130
This means someone must wager $100 to win $130

The weather is also a factor that can influence this money line
Conditions at the beginning of the game are currently cloudy and subject to change.
If it starts raining, scoring is usually less frequent.  
This can affect the money line during the course of the game and is something sports bettors are likely to monitor as they decide where to place their future bets, so it may be a good idea to get this information.

And of course, score information is essential 
We need to be sure to include this information in real time to the users of our app.

To recap, we want to pull the following information for our app

    - Weather conditions in Philadelphia.

    - Scoring information from the Philadelphia-Dallas game.

    - The money line bet for both Philadelphia and Dallas.

These are all things that get streamed on a live basis, so Kafka is perfect for this.  The API's we require should look something like the following:

    - A weather API

    - An NFL scores API

    - A money line API

Now lets say we were to push all of this information into our cluster.  If some subscriber were to try to extract data from our cluster, it would have to sort through a ton of information to sort through to get what it needs.  Maybe we want a lightweight version of our app and our subscriber only needs to pull the money line data, but it has to sort through all of the scores and weather information and waste valuable time doing so.  

This is where the importance of topics comes into play and helps us efficiently organize our flows of data.

For our case scenario, we can set up our topics like so:

    - Philadelphia Weather

    - Philadelphia Score

    - Dallas Score

    - PHI/DAL Money Line

Our Kafka Cluster would look something like this:

![KafkaCluster](https://imgur.com/9czOrGE.gif)

The subscribers to our cluster would pull the relevant information we need.

Our full app would require data from all of the topics.  But suppose we have a subscriber that just needs the money line data.  This is also possible using Kafka, and the flow of data from our topics into our cluster would look something like this.

![ClusterToSubscriber](https://i.imgur.com/Qj9pjnJ.gif)

While subscribers to the cluster can retrieve information from topics, this information is directly consumed and not robustly stored.  This application will include indicators that rely on historical data, but we only have our cluster set up to retrieve live streaming information.  We need a method to implement historical information to from some kind of database to publish to our cluster, and also have some data store that subscribes and stores this information to another database.  

This is where we implement **connectors**.  Connectors are similar in execution to publishers and subscribers, but they **connect** a data store (MongoDB, S3, text files, etc.) to Kafka.  

The **source connector** acts like a publisher, where it takes information from some data store and publishes it to a topic or topics in our Kafka Cluster.

The **sink connector** acts like a subscriber, where it reads the information from some topic(s) in our Kafka Cluster and stores it in a database.  

![Connectors](https://i.imgur.com/ukxDm4F.gif)

We will set up our Kafka server using source and sink connectors in the next section.  This will enable us to use and write data stores in conjunction with live streaming data to provide the full functionality we need.

### Important Notes

While we now understand how this system works on a base level and we can start implementing some logic, there is something very important to keep in mind.  Our system that relies on streaming data must be resilient and avoid failure.  For example if something in our cluster goes down and publishers cannot publish data to our cluster topics, our subscribers wouldn't be able to retreive anything from these topics and this could cause massive problems, especially with financial real time data.  

The project summary mentions that we use Kafka with something called **Zookeeper**, and Zookeeper is responsible for keeping everything in order across our cluster. It manages task assignment to worker nodes, monitors worker node availability, and keeps track of which node is the master node.  If the master node fails, Zookeeper lets the other nodes "race" to become the new master node, and ensures that **only one** wins that race and is locked into that master node position.

Because real world events like thunderstorms or hardware failures can cause havoc across networks, it is essential that Zookeeper is managing our cluster at all times to help us recover from partial failures when they happen and ensure minimum data loss, if any, to our cluster.

When we start the Kafka server in the next section, we will step through how Zookeeper is involved.

### Step 1 - Start the Kafka Server

We need to configure a few parts of our system.

First, we need to make sure that our server is connected to the correct host.

Since we're just using a single computer for now, we are operating on a **standalone** server.

Next, we will need to configure our sink and source connectors, so that our system knows where to draw information from and write information to in our cluster.

The three files we need are the following, found in the **config** directory of Kafka:

    - connect-standalone.properties
    - connect-file-source.properties
    - connect-file-sink.properties

![ConfigFiles](https://i.imgur.com/Nm5533R.png)


**connect-standalone.properties**

The server settings for the standalone server can be found here:

![ServerSettings](https://i.imgur.com/hwtC1ZW.png)

If running this project on a server with a different configuration, such as Hortonworks Sandbox, be sure to update this to the correct settings.

Before we exit, take a look at the following section:

![Converters](https://i.imgur.com/rGeLBr8.png)

Kafka stores its data in what can best be explained as an "ordered, immutable sequence of records".  This log is continually appended as data flows in.  Think of how a Github user continually makes commits to some document in a repository.  The newest version of the file is uploaded, but all older files are still available and cannot be destroyed to ensure resilience.   

That being said, this sequence of records is written in a strict format that Kakfa can understand, and contains a **key**, a **value**, and a **timestamp**.  Kafka cannot automatically detect what the source text is or how to interpret it. 

For example, lets say we have a set of records separated by a | key, and the first record reads John | Smith | 2.9.1987.
Kafka cannot tell which of these is the key and the value, and may view the birthdate as a timestamp and store it like so.  

To avoid any potential problems like this, we need to update the **key.converter** and **value.converter** parameters to reflect the format of data we are using in our cluster.  As the source data for this project is JSON, we do not need to change anything right now.

**connect-file-source.properties**

The first order of business is to determine which file to use as the project source.
Since the default configuration is JSON, we will be abiding by this format for ease of use.
For this example, we will be using **rates.json**, which is available for download in this repository.
This file contains sample quote data for various pairs, along with timestamps that we can write to our database through the sink connector.

Now we are going to need to indicate where our source connector can find the file 
Update the file parameter in the **connect-file-source.properties** file with the location of our source data.

We can also see there is a parameter for **topics**
We didn't need to specify a topic name until now, but we will name our topic **pairs** to reflect the currency pairs data from **rates.json** that we are using for our cluster.
Be sure to update the **topics** parameters to equal **pairs** before proceeding.

**connect-file-sink.properties**

On the other end, we need to update our sink connector properties to listen to the **pairs** topic.
As mentioned before, this connector is also responsible for taking in the data read from the provided topics and writing it to some data store.
As a default setting this file is saved into test.sink.txt, and we can change the file destination as needed.

------------------------------------------------------------------------------------------------------------------


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
