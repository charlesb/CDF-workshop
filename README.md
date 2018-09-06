# HDP/HDF Labs

Content

* [Lab 1 - Accessing the sandbox](#accessing-the-sandbox)
* [Lab 2 - Stream data using NiFi](#stream-data-using-nifi)
* [Lab 3 - Query datasets using Zeppelin](#query datasets using Zeppelin)
* [Lab 4 - Analyze the datasets using SQL](#analyze-the-datasets-using-sql)

## Accessing the sandbox

### Add an alias to your hosts file

On Mac OS X, open a terminal and vi /etc/hosts

On Windows, open C:\Windows\System32\drivers\etc\hosts

Add a new line to the existing

```nn.nnn.nnn.nnn	demo.hortonworks.com```

Replacing the ip (nn.nnn.nnn.nnn) address with the one provided

### Start all HDP and HDF services

Open a web browser and go to the following url

```http://demo.hortonworks.com:8080/```

Log in with the following credential

Username: admin
Password: admin

If services are not started, start all services

![Image of Ambari Start Services](images/start_services.png)

It can take up to 18 minutes...

![Image of Ambari Starting Services](images/starting_services.png)

### SSH to the sandbox

Copy and paste the content of [ppk](keys/hdp-workshop.ppk) for Windows or [pem](keys/hdp-workshop.pem) for Mac OS X

On Mac use the terminal to SSH

For Mac users, don't forget to ```chmod 400 /path/to/hdp-workshop.pem``` before ssh'ing

![Image of Mac terminal ssh](images/mac_terminal_ssh.png)

On Windows use [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

![Image of Putty ssh](images/login_with_putty_1.png)

![Image of Putty ssh](images/login_with_putty_2.png)

## Stream data using NiFi

Open [NiFi](http://demo.hortonworks.com:9090/nifi/) UI

### Get social media sentiment analysis

For the purpose of this exercise we are going to use the [Social Searcher](https://www.social-searcher.com/) monitoring tool.

The API documentation can be found [here](https://www.social-searcher.com/api-v2/)

To get started we need to get the data from the Social Media REST API, extract what we need and save it to a file

We want to have a feeling of the sentiment when people are posting about Hortonworks on Facebook and Twitter

- Step 1: Add a InvokeHTTP processor to the canvas
  - Double click on the processor
  - On settings tab, check all relationships except **Response**
  - On scheduling tab, set Run Schedule to 30 sec
  - Go to properties tab and add the **Remote URL** value: ```https://api.social-searcher.com/v2/search?q=Hortonworks&network=facebook,twitter&limit=20```
  - Apply changes
  
- Step 2: Add a SplitJson connector to the canvas and link from InvokeHttp on **Response** relationship
  - Double click on the processor
  - On settings tab, check both **failure** and **original** relationships
  - On properties tab give **JsonPath Expression** the value: **$.posts**
  - Apply changes
  
- Step 3: Add EvaluateJsonPath to the canvas and link from SplitJson on **split** relationship
  - Double click on the processor
  - On settings tab, check both **failure** and **unmatched** relationships
  - On properties tab
  - Change **Destination** value to **flowfile-attribute**
  - Add properties as follow
    - network: $.network
    - time: $.posted
    - sentiment: $.sentiment
    - text: $.text
    - url: $.url
    
    ![EvaluateJsonPath properties](images/evaluatejsonpathproperties.png)

- Step 4: Add a AttributeToJSON connector to the canvas and link from EvaluateJsonPath on **matched** relationship
  - Double click on the processor
  - On settings tab, check **failure** relationship
  - Change **Destination** value to **flowfile-content**
  - Change **Attribute List** value to **network,time,sentiment,text,url**
  - Apply changes
  
- Step 5: Add a MergeContent connector to the canvas and link from AttributeToJSON on **success** relationship
  - Double click on the processor
  - On settings tab, check both **failure** and **original** relationships
  - Apply changes
  
- Step 6: Add a PutFile connector to the canvas and link from MergeContent on **merge** relationship
  - Double click on the processor
  - On settings tab, check all relationships
  - Change **Directory** value to **/tmp/socialmedia**
  - Change **Attribute List** value to **network,time,sentiment,text,url**
  - Change **Conflict Resolution Strategy** value to **replace**
  - Apply changes
  
- Step 7: Start the entire flowfile

![NiFi Flow 1](images/flow1.png)

Explore the file created under /tmp/socialmedia

## Query datasets using Zeppelin

Visit [Zeppelin](http://demo.hortonworks.com:9995/) 

And log in as admin (password: admin)

su - sudo
cd /usr/hdp/current/kafka-broker
./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic druid_demo

beeline -u "jdbc:hive2://demo.hortonworks.com:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2-interactive" -n admin


set hive.druid.metadata.uri=jdbc:mysql://localhost:3306/druid;

CREATE EXTERNAL TABLE workshop.druid_demo (
`__time` timestamp,
`host` string,
`avgPacketsPerSecond` float,
`avgBytesPerSecond` float
)
STORED BY 'org.apache.hadoop.hive.druid.DruidStorageHandler'
TBLPROPERTIES (
"kafka.bootstrap.servers" = "demo.hortonworks.com:6667",
"kafka.topic" = "druid_demo",
"druid.kafka.ingestion.useEarliestOffset" = "true",
"druid.kafka.ingestion.maxRowsInMemory" = "5",
"druid.kafka.ingestion.startDelay" = "PT1S",
"druid.kafka.ingestion.period" = "PT1S",
"druid.kafka.ingestion.consumer.retries" = "2"
);


ALTER TABLE workshop.druid_demo SET TBLPROPERTIES('druid.kafka.ingestion' = 'START');

cd /usr/hdp/current/kafka-broker

./bin/kafka-console-producer.sh --broker-list demo.hortonworks.com:6667 --topic druid_demo


./bin/kafka-console-consumer.sh --bootstrap-server demo.hortonworks.com:6667 --topic druid_demo


https://api.social-searcher.com/v2/search?q=%22obama%22&network=facebook,twitter&limit=100
https://www.social-searcher.com/api-v2/
https://community.hortonworks.com/articles/193945/social-media-monitoring-with-nifi-hivedruid-integr.html





















