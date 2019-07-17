# Working with Kafka

## Introduction

Streaming integrator could be used to consume from a Kafka topic as well as to publish to a Kafka topic, in a streaming way.

This tutorial takes you through, consuming from a Kafka topic, processing the messages and finally publishing output to a Kafka topic. 

## Tutorial Outline
- [Preparing the server](#preparing-the-server)
- [Starting Kafka](#starting-kafka)
- [Starting Stream Processor](#starting-stream-processor)
- [Consuming from a Kafka topic](#consuming-from-a-kafka-topic)
- [Consuming with an offset](#consuming-with-an-offset)
- [Restoring Offset after system failure](#restoring-offset-after-system-failure)
- [Adding more Consumers to the Consumer Group](#adding-more-consumers-to-the-consumer-group)
- [Assigning Consumers to Partitions](#assigning-consumers-to-partitions)
- [Publishing to a Kafka topic](#publishing-to-a-kafka-topic)


## Preparing the server

Follow these steps to prepare the server in order to consume/publish from/to Kafka.

1. Download the Kafka broker from here: https://www.apache.org/dyn/closer.cgi?path=/kafka/2.3.0/kafka_2.12-2.3.0.tgz and extract it.
We will call the extracted location as {KafkaHome}

2. Create a directory named {Source} in a preferred location in your machine and copy the following JARs to it from the {KafkaHome}/libs directory.
    * kafka_2.12-2.3.0.jar
    * kafka-clients-2.3.0.jar
    * metrics-core-2.2.0.jar
    * scala-library-2.12.8.jar
    * zkclient-0.11.jar
    * zookeeper-3.4.14.jar

3. Create another directory named {Destination} in a preferred location in your machine.

4. Issue following command:
```
sh {WSO2SPHome}/bin/jartobundle.sh <{Source} Directory Path> <{Destination} Directory Path>
```
5. Copy all the jars from the {Destination} directory to {WSO2SPHome}/lib directory.

6. Copy all the jars from the {Source} directory to {WSO2SPHome}/samples/sample-clients/lib directory. 

## Starting Kafka 

1. Navigate to {KafkaHome} and start zookeeper node using "sh bin/zookeeper-server-start.sh config/zookeeper.properties" command.

2. Navigate to {KafkaHome} and start Kafka server node using "sh bin/kafka-server-start.sh config/server.properties" command.

## Starting Stream Processor
Navigate to {WSO2SPHome}/bin directory and issue command "sh worker.sh" 

## Consuming from a Kafka topic

Let's create a basic Siddhi app to consume messages from a Kafka topic.

Open a text file and copy-paste following app into it.

```
@App:name("HelloKafka")

@App:description('Consume events from a Kafka Topic and log the messages on the console.')

@source(type='kafka',
        topic.list='productions',
        threading.option='single.thread',
        group.id="group1",
        bootstrap.servers='localhost:9092',
        @map(type='json'))        
define stream SweetProductionStream (name string, amount double);

@sink(type='log')
define stream OutputStream (name string, amount double);

from SweetProductionStream
select *
insert into OutputStream;
```

Save this file as HelloKafka.siddhi into {WSO2SPHome}/wso2/worker/deployment/siddhi-files directory.

> **_INFO:_**  We just created a Siddhi app which listens to Kafka topic 'productions' and log any incoming messages. However, we still either have not created such a Kafka topic or have pushed any messages to it.  We will do that in the following section.

### Generating Kafka messages

Now let's generate some Kafka messages so that the Stream Processor would receive those. 

First, let's create a topic named "productions" in the Kafka server.

To do that, navigate to {KafkaHome} and run following command:
```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic productions
```
Now let's run the Kafka command line client to push a few messages to the Kafka server.
```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic productions
```
Now you are prompted to type the messages on the console. Type following on the command prompt:
```
{"event":{ "name":"Almond cookie", "amount":100.0}} 
```
This will push a message to Kafka Server which will then be consumed by the Siddhi app we deployed in the Stream Processor. As a result, you should now see following log on the Stream Processor log:
```
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : SweetProductionStream : Event{timestamp=1562069868006, data=[Almond cookie, 100.0], isExpired=false}
```
## Consuming with an offset

Previously, we consumed messages from the topic 'productions' *without specifying an offset*. In other words, the Kafka offset was zero. Rather than consuming with a zero offset, now we will specify an offset value and consume messages from that offset onwards.

For this purpose, we will use the configuration parameter 'topic.offsets.map'.  

Let's modify our previous Siddhi app to specify an offset value. We will specify offset value 2 so that the Siddhi app will consume messages bearing index 2 and above. 

Open HelloKafka.siddhi file, located in {WSO2SPHome}/wso2/worker/deployment/siddhi-files directory and add following new configuration parameter:
``` 
topic.offsets.map='productions=2' 
```
Refer complete siddhi app below:
```
@App:name("HelloKafka")

@App:description('Consume events from a Kafka Topic and log the messages on the console.')

@source(type='kafka',
        topic.list='productions',
        threading.option='single.thread',
        group.id="group1",
        bootstrap.servers='localhost:9092',
        topic.offsets.map='productions=2',
        @map(type='json'))        
define stream SweetProductionStream (name string, amount double);

@sink(type='log')
define stream OutputStream (name string, amount double);

from SweetProductionStream
select *
insert into OutputStream;
```
Save the file.

Now push following message to the Kafka server:
```
{"event":{ "name":"Baked alaska", "amount":20.0}} 
```
Notice that this is the second message that we pushed (hence bearing index 1) and therefore it is not consumed by the stream processor.

Let's push another message (bearing index 2) to the Kafka server:
```
{"event":{ "name":"Cup cake", "amount":300.0}} 
```
Now you will see following log on the Stream Processor. 
```
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562676477785, data=[Cup cake, 300.0], isExpired=false}
```  
As we configured our Siddhi app to consume messages with offset 2, all messages bearing index 2 or above will be consumed.

## Restoring Offset after system failure  

Consider the scenario where the system fails (Stream Processor is shutdown) at the point where the Kafka consumer has consumed upto offset number 5. When the system failure is restored we do not want the Kafka consumer to consume from the beginning (that is offset 0), rather we want the consumption to resume from the point it stopped. To achieve this behaviour, we can use the state persistence capability in the Stream Processor.

If you enable state persistence in the Stream Processor, the Kafka consumer offset will be remembered by the server and upon server restart, the consumption will resume from the point where it stopped. 

To enable state persistence in the Stream Processor, do following.

Open {WSO2SPHome}/conf/worker/deployment.yaml file on a text editor and locate the `state.persistence` section.

``` 
  # Periodic Persistence Configuration
state.persistence:
  enabled: true
  intervalInMin: 1
  revisionsToKeep: 2
  persistenceStore: org.wso2.carbon.stream.processor.core.persistence.FileSystemPersistenceStore
  config:
    location: siddhi-app-persistence
``` 

Set `enabled` option to `true`, save the file and restart the Stream Processor server for this change to be effective.

## Adding more Consumers to the Consumer Group

In our `HelloKafka` Siddhi app, notice the parameter `group.id`. This is used to define the Kafka Consumer Group. 

Now let's modify the Siddhi app to add another Kafka consumer to the same consumer group.

```
@App:name("HelloKafka")

@App:description('Consume events from a Kafka Topic and log the messages on the console.')

@source(type='kafka',
        topic.list='productions',
        threading.option='single.thread',
        group.id="group1",
        bootstrap.servers='localhost:9092',
        @map(type='json'))        
define stream SweetProductionStream1 (name string, amount double);

@source(type='kafka',
        topic.list='productions',
        threading.option='single.thread',
        group.id="group1",
        bootstrap.servers='localhost:9092',
        @map(type='json'))        
define stream SweetProductionStream2 (name string, amount double);

@sink(type='log')
define stream OutputStream (name string, amount double, id string);

from SweetProductionStream1
select name, amount, 'consumer-1' as id 
insert into OutputStream;

from SweetProductionStream2
select name, amount, 'consumer-2' as id 
insert into OutputStream;
```  

Navigate to {KafkaHome} and run following command:
```  
bin/kafka-topics.sh --alter --bootstrap-server localhost:9092 --partitions 2 --topic productions
```  
This will add another partition to the Kafka topic: 'productions'. 

Now push following messages to the Kafka server using the Kafka Console Producer. 
```
{"event":{ "name":"Doughnut", "amount":500.0}} 
```
```
{"event":{ "name":"Danish pastry", "amount":200.0}} 
```
```
{"event":{ "name":"Eclair", "amount":400.0}} 
```
```
{"event":{ "name":"Eclair toffee", "amount":100.0}} 
```
Now observe the Stream Processor logs.
```
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562759480019, data=[Doughnut, 500.0, consumer-2], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562759494710, data=[Danish pastry, 200.0, consumer-1], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562759506252, data=[Eclair, 400.0, consumer-2], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562759508757, data=[Eclair toffee, 100.0, consumer-1], isExpired=false}
```
You could see that the events are being received by the two consumers in a round robin manner. Events received by first consumer can be identified by the id `consumer-1` and similarly, events received by the second consumer can be identified by the id `consumer-2`.

## Assigning Consumers to Partitions  
In the previous scenario, we had two partitions for the Kafka topic and had two consumers. We did not assign the consumers to the partitions, rather we let Kafka do the assignments. Optionally, we can assign consumers to partitions. In case we have multiple consumers which do not perform in an identical way (some consumers might be slow), we might want to use this option and balance the load among the consumers.

Let's alter our topic to have three partitions. After that, we will assign two partitions to consumer-1 and the remaining partition to consumer-2.

Navigate to {KafkaHome} and run following command:
```  
bin/kafka-topics.sh --alter --bootstrap-server localhost:9092 --partitions 3 --topic productions
```  
This will add another partition to the Kafka topic: 'productions' (total of three partitions now). 

To assign partitions to the consumers, we use below Siddhi configuration parameter:
```
partition.no.list
```

Let's modify our `HelloKafka` siddhi app as follows:
```  
@App:name("HelloKafka")

@App:description('Consume events from a Kafka Topic and log the messages on the console.')

-- consumer-1
@source(type='kafka',
        topic.list='productions',
        threading.option='single.thread',
        group.id="group1",
        bootstrap.servers='localhost:9092',
        partition.no.list='0,1',
        @map(type='json'))
define stream SweetProductionStream1 (name string, amount double);

-- consumer-2
@source(type='kafka',
        topic.list='productions',
        threading.option='single.thread',
        group.id="group1",
        bootstrap.servers='localhost:9092',
        partition.no.list='2',
        @map(type='json'))
define stream SweetProductionStream2 (name string, amount double);

@sink(type='log')
define stream OutputStream (name string, amount double, id string);

from SweetProductionStream1
select name, amount, 'consumer-1' as id
insert into OutputStream;

from SweetProductionStream2
select name, amount, 'consumer-2' as id
insert into OutputStream;
```  
Notice that consumer-1 has partitions 0 and 1 assigned; while consumer-2 has partition 2 assigned. 

Now let's publish some messages and see how the load is distributed among the consumers with the new partition assignments. 
```
{"event":{ "name":"Fortune cookie", "amount":100.0}} 
```
```
{"event":{ "name":"Frozen yogurt", "amount":350.0}} 
```
```
{"event":{ "name":"Gingerbread", "amount":450.0}} 
```
```
{"event":{ "name":"Hot-fudge sundae", "amount":150.0}} 
```
```
{"event":{ "name":"Hot-chocolate pudding", "amount":200.0}} 
```
```
{"event":{ "name":"Ice cream cake", "amount":250.0}} 
```
Now observe the Stream Processor logs.
```
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562851086792, data=[Fortune cookie, 100.0, consumer-1], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562851092100, data=[Frozen yogurt, 350.0, consumer-1], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562851094459, data=[Gingerbread, 450.0, consumer-2], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562851096434, data=[Hot-fudge sundae, 150.0, consumer-1], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562851098328, data=[Hot-chocolate pudding, 200.0, consumer-1], isExpired=false}
INFO {org.wso2.siddhi.core.stream.output.sink.LogSink} - HelloKafka : OutputStream : Event{timestamp=1562851100309, data=[Ice cream cake, 250.0, consumer-2], isExpired=false}
```
You could observe the pattern that the load is distributed among consumer-1 and consumer-2 in 2:1 ratio. This is because we assigned two partitions to consumer-1 and assigned only one partition to consumer-2. 

## Publishing to a Kafka topic

We started this tutorial with a Siddhi app to consume from a Kafka topic and log the incoming messages. Now we will create a new Siddhi app to consume from `productions` topic, filter the incoming messages based on a condition and then, to publish those filtered messages to another Kafka topic. 
'
First, let's create a new topic named "bulk-orders" in the Kafka server. We will publish the filtered messages to this Kafka topic.

To do that, navigate to {KafkaHome} and run following command:
```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic bulk-orders
```
Next, let's create the Siddhi app.

Open a text file and copy-paste following app into it.

```
@App:name("PublishToKafka")

@App:description('Consume events from a Kafka Topic, do basic filtering and publish filtered messages to a Kafka topic.')

@source(type='kafka',
        topic.list='productions',
        threading.option='single.thread',
        group.id="group2",
        bootstrap.servers='localhost:9092',
        @map(type='json'))        
define stream SweetProductionStream (name string, amount double);

@sink(type='kafka',
      topic='bulk-orders',
      bootstrap.servers='localhost:9092',
      partition.no='0',
      @map(type='json'))
define stream BulkOrdersStream (name string, amount double);

from SweetProductionStream[amount > 100]
select *
insert into BulkOrdersStream;
```
Save this file as PublishToKafka.siddhi into {WSO2SPHome}/wso2/worker/deployment/siddhi-files directory. Upon successful deployment of the Siddhi app, you will see following INFO log on the Stream Processor console.
```
INFO {org.wso2.carbon.stream.processor.core.internal.StreamProcessorService} - Siddhi App PublishToKafka deployed successfully
```
> **_INFO:_**  The Siddhi app `PublishToKafka` consumes all the messages from the topic `productions` and populates the stream `SweetProductionStream`. All the sweet productions having the amount greater than 100 will be inserted into stream `BulkOrdersStream` and those events will be pushed to Kafka topic `bulk-orders`. 

Now to observe the messages in the `bulk-orders` topic, we will run a Kafka Console Consumer. Navigate to {KafkaHome} and run following command:
```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bulk-orders --from-beginning
```
You will see following messages on the Consumer log, which are the productions having amount greater than 100:
```
{"event":{ "name":"Cup cake", "amount":300.0}}
{"event":{ "name":"Doughnut", "amount":500.0}} 
{"event":{ "name":"Danish pastry", "amount":200.0}} 
{"event":{ "name":"Eclair", "amount":400.0}}  
{"event":{ "name":"Frozen yogurt", "amount":350.0}} 
{"event":{ "name":"Hot-fudge sundae", "amount":150.0}}
{"event":{ "name":"Hot-chocolate pudding", "amount":200.0}} 
{"event":{ "name":"Ice cream cake", "amount":250.0}}
``` 
