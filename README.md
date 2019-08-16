# CDF Labs: Real-time sentiment analysis with NiFi, Kafka, Druid, DAS and Superset


## Prerequisite

**Although this AMI is not public and is available for Cloudera workhops only, the steps can be reproduced in your own environment**

- Launch AWS AMI **ami-009b8088cc86ad145** in Asia Pacific Singpore region (ap-southeast-1) with **m5d.4xlarge** instance type
- Keep default storage (300GB SSD)
- Set security group with:
  - Type: All TCP
  - Source: My IP
- Choose an existing or create a new key pair

## Content

* [Lab 1 - Accessing the sandbox](#accessing-the-sandbox)
* [Lab 2 - Stream data using NiFi](#stream-data-using-nifi)
* [Lab 3 - Explore Kafka](#explore-kafka)
* [Lab 4 - Integrate with Schema Registry](#integrate-with-schema-registry)
* [Lab 5 - Explore DAS, Hive and Druid](#explore-das-hive-and-druid)
* [Lab 6 - Stream enhanced data into Hive using NiFi](#stream-enhanced-data-into-hive-using-nifi)
* [Lab 7 - Create live dashboard with Superset](#create-live-dashboard-with-superset)
* [Lab 8 - Collect syslog data using MiNiFi and EFM](#collect-syslog-data-using-minifi-and-efm)
* [Bonus - Process sentiment analysis on tweets](#process-sentiment-analysis-on-tweets)

## Accessing the sandbox

### Add an alias to your hosts file

On Mac OS X, open a terminal and vi /etc/hosts

On Windows, open C:\Windows\System32\drivers\etc\hosts

Add a new line to the existing

```nn.nnn.nnn.nnn	demo.cloudera.com```

Replacing the ip (nn.nnn.nnn.nnn) address with the one provided

If you can't edit the hosts file due to lack of privileges, then you will need to replace the reference to demo.cloudera.com alias with the instance private ip wherever it's used by a NiFi processor.

To get this private ip, ssh to the instance and type the command ```ifconfig```, the first ip starting with 172 is the one to use:

![Private IP](images/private-ip.png)

### Start all HDP and CDF services

**They should be already started**

Open a web browser and go to the following url

```http://demo.cloudera.com:8080/```

Log in with the following credential

Username: admin
Password: admin

If services are not started already, start all services

![Image of Ambari Start Services](images/start_services.png)

It can take up to 20 minutes...

![Image of Ambari Starting Services](images/starting_services.png)

### SSH to the sandbox

**Copy and paste** (do not download) the content of [ppk](https://raw.githubusercontent.com/charlesb/CDF-workshop/master/keys/hdp-workshop.ppk) for Windows or [pem](https://raw.githubusercontent.com/charlesb/CDF-workshop/master/keys/hdp-workshop.pem) for Mac OS X and save it locally.

On Mac use the terminal to SSH

For Mac users, don't forget to ```chmod 400 /path/to/hdp-workshop.pem``` before ssh'ing

![Image of Mac terminal ssh](images/mac_terminal_ssh.png)

On Windows use [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

![Image of Putty ssh](images/login_with_putty_1.png)

![Image of Putty ssh](images/login_with_putty_2.png)

## Stream data using NiFi

### Build NiFi flow

In order to have a streaming source available for our workshop, we are going to make use of the publicly available Meetup's API and connect to their WebSocket.

The API documentation is available [here](https://www.meetup.com/meetup_api/docs/stream/2/event_comments/#websockets): https://www.meetup.com/meetup_api/docs/stream/2/event_comments/#websockets

In this workshop we are going to stream all comments, for all topics, into NiFi and classify each one of them into the 5 categories listed below:

- very negative
- negative
- neutral
- positive
- very positive 

To do so we will score each comment against the Stanford CoreNLP's sentiment model as we will see [later](https://github.com/charlesb/CDF-workshop#run-the-sentiment-analysis-model-as-a-rest-like-service). 

In real-world use case we would probably filter by event of our interest but for the sake of this workshop we won't and assume all comments are given for the same event: the famous CDF workshop!

Let's get started... Open [NiFi UI](http://demo.cloudera.com:9090/nifi/) and follow the steps below:

- Step 1: Drag on drop a Process Group on the root canvas and name it **CDF Workshop**

![CDF Workshop process group](images/cdfprocessgroup.png)

- Step 2: Go to [NiFi Registry](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) and create a new bucket
  - Click on the little wrench icon at the top right corner
  - Click on the **NEW BUCKET** button
  - Name the bucket **workshop**
  
![NiFi Registry bucket creation](images/registry-bucket.png)

- Step 3: Go back to NiFi UI and right click on the previously created process group
  - Click on Version > Start version control
  - Then provide at least a Flow Name
  - Click on Save

![Version control](images/version-control.png)

![Version control 2](images/version-control-2.png)

- Step 4: Get in the CDF Workshop (double click on the process group) and add a **ConnectWebSocket** processor to the canvas
  - Double click on the processor
  - On settings tab, check all relationships except **text message**
  - Got to properties tab and select or create **JettyWebSocketClient** as the WebSocket Client ControllerService
  - Then configure the service (click on the arrow on the right)	
  	- Go to properties tab and add this value: ```ws://stream.meetup.com/2/event_comments``` to property **WebSocket URI**
  	- Apply the change
  	- Enable the controller service (click on the thunder icon) and close the window
  - Go to properties tab and give a value to **WebSocket Client Id** such as **demo** for example
  - Apply changes
  
- Step 5: Add an UpdateAttribute connector to the canvas and link from ConnectWebSocket on **text message** relationship
  - Double click on the processor
  - On properties tab add new property **mime.type** clicking on + icon and give the value **application/json**. This will tell the next processor that the messages sent by the Meetup WebSocket is in JSON format.
  - Add another property **event** to set an event name **CDF workshop** for the purpose of this exercise as explained before
  - Apply changes
  
![UpdateAtrribute 1 properties](images/updateattibute1properties.png)
  
- Step 6: Add EvaluateJsonPath to the canvas and link from UpdateAttribute on **success** relationship
  - Double click on the processor
  - On settings tab, check both **failure** and **unmatched** relationships
  - On properties tab, change **Destination** value to **flowfile-attribute**
  - And add properties as follow
    - comment: $.comment
    - member: $.member.member_name
    - timestamp: $.mtime
    - country: $.group.country
    
![EvaluateJsonPath 1 properties](images/evaluatejsonpathproperties1.png)
    
    The messages coming out of the web sockets look like this:
    
    ```json
    {"visibility":"public","member":{"member_id":11643711,"photo":"https:\/\/secure.meetupstatic.com\/photos\/member\/3\/1\/6\/8\/thumb_273072648.jpeg","member_name":"Loka Murphy"},"comment":"I didn’t when I registered but now thinking I want to try and get one since it’s only taking place once.","id":-259414201,"mtime":1541557753087,"event":{"event_name":"Tunnel to Viaduct 8k Run","event_id":"256109695"},"table_name":"event_comment","group":{"join_mode":"open","country":"us","city":"Seattle","name":"Seattle Green Lake Running Group","group_lon":-122.34,"id":1608555,"state":"WA","urlname":"Seattle-Greenlake-Running-Group","category":{"name":"fitness","id":9,"shortname":"fitness"},"group_photo":{"highres_link":"https:\/\/secure.meetupstatic.com\/photos\/event\/9\/e\/f\/4\/highres_465640692.jpeg","photo_link":"https:\/\/secure.meetupstatic.com\/photos\/event\/9\/e\/f\/4\/600_465640692.jpeg","photo_id":465640692,"thumb_link":"https:\/\/secure.meetupstatic.com\/photos\/event\/9\/e\/f\/4\/thumb_465640692.jpeg"},"group_lat":47.61},"in_reply_to":496130460,"status":"active"}
    ```

- Step 7: Add an AttributesToCSV processor to the canvas and link from EvaluateJsonPath on **matched** relationship
  - Double click on the processor
  - On settings tab, check **failure** relationship
  - Change **Destination** value to **flowfile-content**
  - Change **Attribute List** value to write only the above parsed attributes: **timestamp,event,member,comment,country**
  - Set Include Core Attributes to **false**
  - Set **Include Schema** to **true**
  - Apply changes
  
- Step 8: Add a PutFile processor to the canvas and link from AttributesToCSV on **success** relationship
  - Double click on the processor
  - On settings tab, check all relationships
  - Change **Directory** value to **/tmp/workshop**
  - Apply changes

- Step 9: Right-click anywhere on the canvas and commit your first flow!

![NiFi commit 1](images/commit-1.png)
![NiFi commit 2](images/commit-2.png)

If you visit the [NiFi Registry UI](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) again you should see your commit.

![NiFi commit 3](images/commit-3.png)

- Step 10: Start the entire flow

![NiFi Flow 1](images/flow1.png)

SSH to the sandbox and explore the files created under /tmp/workshop.

On the NiFi UI, explore the FlowFiles' attributes and content looking at Data provenance.

**Once done, stop the flow and delete all files ```sudo rm -rf /tmp/workshop/*```**

## Explore Kafka

ssh to the AWS instance as explained above then become root

```sudo su -```

Navigate to Kafka

```cd /usr/hdp/current/kafka-broker```

Create a topic named **meetup_comment_ws**

```./bin/kafka-topics.sh --create --zookeeper demo.cloudera.com:2181 --replication-factor 1 --partitions 1 --topic meetup_comment_ws```

List topics to check that it's been created

```./bin/kafka-topics.sh --list --zookeeper demo.cloudera.com:2181```

Open a consumer so later we can monitor and verify that JSON records will stream through this topic:

```./bin/kafka-console-consumer.sh --bootstrap-server demo.cloudera.com:6667 --topic meetup_comment_ws```

Keep this terminal open.

We will now open a new terminal to publish some messages...

Follow the same steps as above except for the last step where we are going to open a producer instead of a consumer:

```./bin/kafka-console-producer.sh --broker-list demo.cloudera.com:6667 --topic meetup_comment_ws```

Type anything and click enter. Then go back to the first terminal with the consumer running. You should see the same message get displayed!

## Integrate with Schema Registry

Explore [Schema Registry UI](http://demo.cloudera.com:7788/)

Create a new Avro Schema, hitting the plus button, named **meetup_comment_avro** with the following Avro Schema:

```Avro
{
     "type": "record",
     "name": "meetup_comment_avro",
     "fields": [
       {"name": "country", "type": "string"},
       {"name": "member", "type": "string"},
       {"name": "comment", "type": "string"},
       {"name": "event", "type": "string"},
       {"name": "timestamp", "type": "long"}
       ]
}
```

![Avro schema creation](images/avro_schema_creation.png)

You should end up with a newly versioned schema as follow:

![Avro schema versioned](images/avro_schema_versioned.png)

Explore the [REST API](http://demo.cloudera.com:7788/swagger) as well.

Remove the last processor PutFile as we are going to stream the avro record to some Kafka topic.

Now we are going to filter the records we are interested in and convert them from CSV to Avro in the process.

- Step 1: Remove the existing PutFile processor and add a QueryRecord processor to the canvas and link from AttributesToCSV on **success** relationship
  - Double click on the processor
  - On properties tab
  	- For RecordReader, create a CSVReader service
  	- For RecordWrite, create a AvroRecordSetWriter service
  	- Configure both services
  	  - For CSVReader, use String Fields From Header for the Schema Access Strategy
  	  - For AvroRecordSetWriter, we are going to connect to the Schema Registry API (http://demo.cloudera.com:7788/api/v1) and use the avro schema created before. For the **Schema Registry** property choose **HortonworksSchemaRegistry** as shown in the screen shot below and configure it.
  	    - For **Schema Access Strategy** use 'Schema Name' property
  	    - For **Schema Name** property add ```meetup_comment_avro```
  	- Filter comments per country as our sentiment analysis model supports English only
  	  - Add a property **comments_in_english** with value ```SELECT * FROM FLOWFILE WHERE country IN ('gb', 'us', 'sg')```
  	- Set **Include Zero Record FlowFiles** to false
  - On settings tab, check **failure** and **original** relationships
  - Apply changes
  
![Query Record](images/query_record.png)

![CSVReader](images/csv_reader.png)

![AvroRecordSetWriter](images/avro_record_set_writer.png)

Set the HortonworksSchemaRegistry controller service as follow

![HWXSchemaRegistry](images/hwx_schema_registry.png)

Enable all controller services.

- Step 2: Add a **PublishKafka_2_0** connector to the canvas and link from QueryRecord on **comments_in_english** relationship
  - Double click on the processor
  - On settings tab, check all relationships
  - On properties tab
  - Change **Kafka Brokers** value to **demo.cloudera.com:6667**
  - Change **Topic Name** value to **meetup_comment_ws**
  - Change **Use Transactions** value to **false**
  - Apply changes

The flow should look like this:

![Avro records to Kafka topic](images/avro_records_to_kafka_topic.png)

Again commit your changes and start the flow!

You should be able to see records streaming through Kafka looking at the terminal with Kafka consumer opened earlier

![Kafka topic avro](images/kafka_topic_avro.png)

When you are happy with the outcome stop the flow and purge the Kafka topic as we are going to use it later:

```./bin/kafka-configs.sh --zookeeper demo.cloudera.com:2181 --entity-type topics --alter --entity-name meetup_comment_ws --add-config retention.ms=1000```

Wait for few second and set the retention back to one hour:

```./bin/kafka-configs.sh --zookeeper demo.cloudera.com:2181 --entity-type topics --alter --entity-name meetup_comment_ws --add-config retention.ms=3600000```

You can check if the retention was set properly:

```./bin/kafka-configs.sh --zookeeper demo.cloudera.com:2181 --describe --entity-type topics --entity-name meetup_comment_ws```

## Explore DAS, Hive and Druid

Before we implement the next workflow, we need to create one Hive external table and start the corresponding Druid source indexing

Visit [Data Analytics Studio (DAS)](http://demo.cloudera.com:30800/)

Click on the **COMPOSE QUERY** button and run the queries below

![DAS Query](images/das-worksheet.png)

Create a database named workshop and run the SQL

```SQL
CREATE DATABASE workshop;
```

Create the Hive table backed by Druid storage where the social medias sentiment analysis will be streamed into

```SQL
CREATE EXTERNAL TABLE workshop.meetup_comment_sentiment (
`__time` timestamp,
`event` string,
`member` string,
`comment` string,
`sentiment` string
)
STORED BY 'org.apache.hadoop.hive.druid.DruidStorageHandler'
TBLPROPERTIES (
"kafka.bootstrap.servers" = "demo.cloudera.com:6667",
"kafka.topic" = "meetup_comment_ws",
"druid.kafka.ingestion.useEarliestOffset" = "true",
"druid.kafka.ingestion.maxRowsInMemory" = "5",
"druid.kafka.ingestion.startDelay" = "PT1S",
"druid.kafka.ingestion.period" = "PT1S",
"druid.kafka.ingestion.consumer.retries" = "2"
);
```
Start Druid indexing

```SQL
ALTER TABLE workshop.meetup_comment_sentiment SET TBLPROPERTIES('druid.kafka.ingestion' = 'START');
```

![DAS show tables](images/das-show-tables.png)


Verify that supervisor and indexing task are running from the [Druid overload console](http://demo.cloudera.com:8090/console.html)

![Druid console](images/druid_console.png)

## Stream enhanced data into Hive using NiFi

### Run the sentiment analysis model as a REST-like service

For the purpose of this exercise we are not going to train, test and implement a classification model but re-use an existing sentiment analysis model, provided by the Stanford University as part of their [CoreNLP - Natural language software](https://stanfordnlp.github.io/CoreNLP/)

First, after ssh'ing to the sandbox, download and unzip the CoreNLP using the wget as below:

```bash
wget http://nlp.stanford.edu/software/stanford-corenlp-full-2018-10-05.zip
unzip stanford-corenlp-full-2018-10-05.zip
```

Then, in order to start the [web service](https://stanfordnlp.github.io/CoreNLP/corenlp-server.html), run the [CoreNLP jar file](https://stanfordnlp.github.io/CoreNLP/download.html), with the following commands:

```bash
cd stanford-corenlp-full-2018-10-05
java -mx1g -cp "*" edu.stanford.nlp.pipeline.StanfordCoreNLPServer -port 9999 -timeout 15000 </dev/null &>/dev/null &
```

This will run in the background on port 9999 and you can visit the [web page](http://demo.cloudera.com:9999/) to make sure it's running.

If you want to play with it, remove all annotations and use **Sentiment** only

![Image of CoreNLP web service](images/corenlp_web.png)

The model will classify the given text into 5 categories:

- very negative
- negative
- neutral
- positive
- very positive

### Enhance the Meetup comments with sentiment analysis outcome 

Go back to [NiFi UI](http://demo.cloudera.com:9090/nifi/) and follow the steps below. Between the QueryRecord and PublishKafka_2_0 processors we are going to enhance the Meetup comment with NLP.

- Step 1: Prepare the content to be posted to the sentiment analysis service
  - Add ReplaceText processor and link from QueryRecord on **comments_in_english** relationship
  - Keep the PublishKafka_2_0 processor on the canvas unlinked from/to other processors for now
  - Double click on processor and check **failure** on settings tab
  - Go to properties tab and remove value for **Search Value** and set it to empty string
  - Set **Replacement Value** with value: **${comment:replaceAll('\\.', ';')}**. We want to make sure the entire comment is evaluated as one sentence instead of one evaluation per sentence within the same comment.
  - Set **Replacement Strategy** to **Always Replace**
  - Apply changes
  
- Step 2: Call the web service started earlier on incoming message
  - Add InvokeHTTP processor and link from ReplaceText on **success** relationship
  - Double click on processor and check all relationships except **Response** on settings tab
  - Go to properties tab and set value for **HTTP Method** to **POST**
  - Set **Remote URL** with value: ```http://demo.cloudera.com:9999/?properties=%7B%22annotators%22%3A%22sentiment%22%2C%22outputFormat%22%3A%22json%22%7D``` which is the url encoded value for **http://demo.cloudera.com:9999/?properties={"annotators":"sentiment","outputFormat":"json"}**
  - Set **Content-Type** to **application/x-www-form-urlencoded**
  - Apply changes
  
- Step 3: Add EvaluateJsonPath to the canvas and link from InvokeHTTP on **Response** relationship
  - Double click on the processor
  - On settings tab, check both **failure** and **unmatched** relationships
  - On properties tab
  - Change **Destination** value to **flowfile-attribute**
  - Add on the property **sentiment** with value **$.sentences[0].sentiment**
  - Apply changes
  
- Step 4: Format post time to comply with [ISO format](https://en.wikipedia.org/wiki/ISO_8601) (Druid requirement)
  - Add UpdateAttribute processor and link from EvaluateJsonPath on **matched** relationship
  - Using handy [NiFi's language expression](https://nifi.apache.org/docs/nifi-docs/html/expression-language-guide.html#dates), add a new attribue ```__time``` with value: ```${timestamp:format("yyyy-MM-dd'T'HH:mm:ss'Z'", "Asia/Singapore")}``` to properties tab

- Step 5: Add AttributesToJSON processor to prepare the message to be published to the Kafka topic created before. Link from UpdateAttribute.
  - Double click on processor
  - On settings tab, check **failure** relationship
  - Go to properties tab
  - In the Attributes List value set ```__time, event, comment, member, sentiment``` to match the previously created Hive table
  - Change Destination to **flowfile-content**
  - Set Include Core Attributes to **false**
  - Apply changes
  
- Step 6: Last step, link existing **PublishKafka_2_0** connector to the canvas from AttributesToJSON on **success** relationship
  
Before starting the NiFi flow make sure that the [sentiment analysis Web service](https://github.com/charlesb/CDF-workshop#run-the-sentiment-analysis-model-as-a-rest-like-service) is running
	
The overall flow should look like this

![NiFi Flow 2](images/nifi_stream_to_kafka.png)

You should be able to see records streaming through Kafka looking at the terminal with Kafka consumer opened earlier

![Kafka topic consumer](images/kafka_topic_consumer.png)

Going back to DAS, we can query the data streamed in real-time

![DAS Query 1](images/das-query-1.png)

![DAS Query 2](images/das-query-2.png)

## Create live dashboard with Superset

Go to [Superset UI](http://demo.cloudera.com:9088/)

Log in with user **admin** and password **admin**

Refresh Druid metadata

![Refresh druid metadata](images/superset_refresh_datasources.png)

Edit the **meetup_comment_sentiment** datasource record and verify that the columns are listed, same for the metric (you might need to scroll down)

![Druid datasource columns](images/druid_datasource_columns.png)

Click on the datasource and create the following query

![Druid query](images/druid_query.png)

From this query, create a dashboard that will refresh automatically

![Druid dashboard](images/druid_dashboard.png)

## Collect syslog data using MiNiFi and EFM

Visit [EFM UI](http://demo.cloudera.com:10080/efm/ui/)

![EFM UI](images/efm-ui.png)

Click on the **EVENTS** tab. List is empty as we havent deployed any MiNiFi agent yet.

![EFM Event 1](images/efm-event-1.png)

A MiNiFi C++ archive (nifi-minifi-cpp-centos-0.7.0-bin.tar.gz) as been uploaded to **/home/centos** already. Deploy it, configure and start it as follow:

```bash
cd /home/centos
tar -xvf nifi-minifi-cpp-centos-0.7.0-bin.tar.gz
cd nifi-minifi-cpp-0.7.0
mv conf/minifi.properties conf/minifi.properties.bk
vim conf/minifi.properties
```

And copy/paste this [content](https://raw.githubusercontent.com/charlesb/CDF-workshop/master/minifi/minifi.properties)):

```properties
# Core Properties #
nifi.version=0.7.0
nifi.flow.configuration.file=./conf/config.yml
nifi.administrative.yield.duration=30 sec
# If a component has no work to do (is "bored"), how long should we wait before checking again for work?
nifi.bored.yield.duration=10 millis

# Provenance Repository #
nifi.provenance.repository.directory.default=/home/centos/nifi-minifi-cpp-0.7.0/provenance_repository
nifi.provenance.repository.max.storage.time=1 MIN
nifi.provenance.repository.max.storage.size=1 MB
nifi.flowfile.repository.directory.default=/home/centos/nifi-minifi-cpp-0.7.0/flowfile_repository
nifi.database.content.repository.directory.default=/home/centos/nifi-minifi-cpp-0.7.0/content_repository

#nifi.remote.input.secure=true
#nifi.security.need.ClientAuth=
#nifi.security.client.certificate=
#nifi.security.client.private.key=
#nifi.security.client.pass.phrase=
#nifi.security.client.ca.certificate=

#nifi.rest.api.user.name=admin
#nifi.rest.api.password=password

## Enabling C2 Uncomment each of the following options
## define those with missing options
nifi.c2.enable=true
## define protocol parameters
## The default is CoAP, if that extension is built. 
## Alternatively, you may use RESTSender if http-curl is built
nifi.c2.agent.protocol.class=RESTSender
#nifi.c2.agent.coap.host=
#nifi.c2.agent.coap.port=
## base URL of the c2 server,
## very likely the same base url of rest urls
#nifi.c2.flow.base.url=
nifi.c2.rest.url=http://demo.cloudera.com:10080/efm/api/c2-protocol/heartbeat
nifi.c2.rest.url.ack=http://demo.cloudera.com:10080/efm/api/c2-protocol/acknowledge
nifi.c2.root.classes=DeviceInfoNode,AgentInformation,FlowInformation
## heartbeat 4 times a second
nifi.c2.agent.heartbeat.period=1000
## define parameters about your agent 
nifi.c2.agent.class=cdfws
nifi.c2.agent.identifier=cdfws
## define metrics reported
nifi.c2.root.class.definitions=metrics
nifi.c2.root.class.definitions.metrics.name=metrics
nifi.c2.root.class.definitions.metrics.metrics=typedmetrics
nifi.c2.root.class.definitions.metrics.metrics.typedmetrics.name=RuntimeMetrics
nifi.c2.root.class.definitions.metrics.metrics.queuemetrics.name=QueueMetrics
nifi.c2.root.class.definitions.metrics.metrics.queuemetrics.classes=QueueMetrics
nifi.c2.root.class.definitions.metrics.metrics.typedmetrics.classes=ProcessMetrics,SystemInformation
nifi.c2.root.class.definitions.metrics.metrics.processorMetrics.name=ProcessorMetric
nifi.c2.root.class.definitions.metrics.metrics.processorMetrics.classes=GetFileMetrics

## enable the controller socket provider on port 9998
## off by default. C2 must be enabled to support these
controller.socket.host=demo.cloudera.com
controller.socket.port=9998


#JNI properties
nifi.framework.dir=/home/centos/nifi-minifi-cpp-0.7.0/minifi-jni/lib
nifi.nar.directory=/home/centos/nifi-minifi-cpp-0.7.0/minifi-jni/nars
nifi.nar.deploy.directory=/home/centos/nifi-minifi-cpp-0.7.0/minifi-jni/nardeploy
nifi.nar.docs.directory=/home/centos/nifi-minifi-cpp-0.7.0/minifi-jni/nardocs
# must be comma separated 
nifi.jvm.options=-Xmx1G
nifi.python.processor.dir=/home/centos/nifi-minifi-cpp-0.7.0/minifi-python/

c2.agent.heartbeat.reporter.classes=RESTReceiver
```

Then start MiNiFi agent with sudo using the command below:

```bash
sudo ./bin/minifi.sh start
```

The EFM UI Events tab should show some heartbeats now

![EFM Event 1](images/efm-event-2.png)

Before we create a flow we need to create a bucket

Go to [NiFi Registry](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) and create a bucket named **efm_demo**

Now, **on the NiFi root canvas**, create a simple flow to collect local syslog messages and forward them to NiFi, where the logs will be parsed, transformed into another format and pushed to a Kafka topic.

Our agent has been tagged with the class 'cdfws' so we are going to create a template under this specific class.

But first we need to add an Input Port to the root canvas of NiFi and build a flow as described before. Input Port are used to receive flow files from remote MiNiFi agents or other NiFi instances.

![NiFi syslog parser](images/nifi-syslog-parser.png)

Don't forget to create a new Kafka topic as explained in Lab 3 above.

We are going to use a Grok parser to parse the syslog messages. Here is a Grok expression that can be used to parse such logs format:

```%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}```

Now that we have built the NiFi flow that will receive the logs, let's go back to the EFM UI and build the MiNiFi flow as below:

![CEM flow](images/cem-minifi-flow.png)

This MiNiFi agent will tail /var/log/messages and send the logs to a remote process group (our NiFi instance) using the Input Port.

![Tailfile](images/tail-file.png)

Please note that the NiFi instance has been configured to receive data over HTTP only, not RAW

![Remote process group](images/remote-process-group.png)

Now we can start the NiFi flow and publish the MiNiFi flow to NiFi registry (Actions > Publish...)

Visit [NiFi Registry UI](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) to make sure your flow has been published successfully.

![NiFi Registry](images/nifi-registry.png)

Within few seconds, you should be able to see syslog messages streaming through your NiFi flow and be published to the Kafka topic you have created.

![Syslog message](images/syslog-json.png)

## Process sentiment analysis on tweets

### Apply for Twitter developer account and create an app

Visit [Twitter developer page](https://developer.twitter.com/en/apply-for-access.html) and click on Apply for a developer account. If you don't have a Twitter account, sign up.

After you have added a valid email address, follow the different account creation steps

![twitter-account-details](images/twitter-account-details.png)

![twitter-usecase-details](images/twitter-usecase-details.png)

For the use case details, you can reuse the text below:

```
1. This account will be used for demo, building streaming data flow with real-time sentiment analyses using NiFi, Kafka and Druid
2. I intend to compare tweets against a machine learning model using NLP techniques for sentiment analysis
3. My use case does not involve tweeting, retweeting or liking content
4. Individual tweets will not be displayed, data will be aggregated per sentiment: very negative, negative, neutral, positive and very positive
```

Agree to the Terms of Services

![twitter-termsofservices-details](images/twitter-termsofservices-details.png)

Finally click on the link from the email you have received and create an app

![twitter-createanapp](images/twitter-createanapp.png)

![twitter-appdetails](images/twitter-appdetails.png)

Finally create the Keys and Tokens that will be needed by the NiFi processor to pull Tweets using the Twitter API

![twitter-keysandtokens](images/twitter-keysandtokens.png)

## Create a NiFi flow

- Step 1: Get the tweets to be analysed
  - Add GetTwitter processor to the canvas
  - **Important!** From Scheduling tab change Run Schedule property to **2 sec**
  - On the Properties tab
    - Set Twitter Endpoint to **Filter Endpoint**
    - Fill the Consumer Key, Consumer Secret, Access Token and Access Token Secret with the app keys and token
    - Set Languages to **en**
    - Provide a set of Terms to Filter On, i.e. 'cloudera'
  - Apply changes
  
- Step 2: Call the web service started earlier on incoming message
  - Add InvokeHTTP processor and link from ReplaceText on **success** relationship
  - Double click on processor and check all relationships except **Response** on settings tab
  - Go to properties tab and set value for **HTTP Method** to **POST**
  - Set **Remote URL** with value: ```http://demo.cloudera.com:9999/?properties=%7B%22annotators%22%3A%22sentiment%22%2C%22outputFormat%22%3A%22json%22%7D``` which is the url encoded value for **http://demo.cloudera.com:9999/?properties={"annotators":"sentiment","outputFormat":"json"}**
  - Set **Content-Type** to **application/x-www-form-urlencoded**
  - Apply changes












