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

Download either the [ppk](keys/hdp-workshop.ppk) or [pem](keys/hdp-workshop.pem) private key whether you are using Windows or Mac

On Mac use the terminal to SSH

For Mac users, don't forget to ```chmod 400 /path/to/hdp-workshop.ppk``` before ssh'ing

![Image of Mac terminal ssh](images/mac_terminal_ssh.png)

On Windows use [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

![Image of Putty ssh](images/login_with_putty_1.png)

![Image of Putty ssh](images/login_with_putty_2.png)

## Stream data using NiFi

Visit [NiFi](http://demo.hortonworks.com:9090/nifi/)

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





















