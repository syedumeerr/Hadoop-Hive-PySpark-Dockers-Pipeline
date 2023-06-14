# ChicagoCrimePrediction

EDA and Predictive Modelling on the Chicago Crime Dataset using PySpark, Hive and storage using HDFS as part of a Big Data project.

**Dataset:** https://data.cityofchicago.org/Public-Safety/Crimes-2001-to-Present/ijzp-q8t2/data

**Abstract:** We have procured criminal records from the Chicago Police Department’s CLEAR (Citizen Law Enforcement Analysis and Reporting) system. To accomplish our objective of recognizing crime patterns across the city based on geographical locations, we have used Hadoop and Apache Spark to achieve faster data processing and provide near real-time predictive analytics on top of that. The attributes (explained above) comprise the dataset of the system model adopted by our project and will be conducive while plotting the exact locations of the crimes.


# Docker multi-container environment with Hadoop, Spark and Hive

This is it: a Docker multi-container environment with Hadoop (HDFS), Spark and Hive. But without the large memory requirements of a Cloudera sandbox. (On my Windows 10 laptop (with WSL2) it seems to consume a mere 3 GB.)

The only thing lacking, is that Hive server doesn't start automatically. To be added when I understand how to do that in docker-compose.


## Quick Start

To deploy an the HDFS-Spark-Hive cluster, run:
```
  docker-compose up
```

`docker-compose` creates a docker network that can be found by running `docker network list`, e.g. `docker-hadoop-spark-hive_default`.

Run `docker network inspect` on the network (e.g. `docker-hadoop-spark-hive_default`) to find the IP the hadoop interfaces are published on. Access these interfaces with the following URLs:

* Namenode: http://<dockerhadoop_IP_address>:9870/dfshealth.html#tab-overview
* History server: http://<dockerhadoop_IP_address>:8188/applicationhistory
* Datanode: http://<dockerhadoop_IP_address>:9864/
* Nodemanager: http://<dockerhadoop_IP_address>:8042/node
* Resource manager: http://<dockerhadoop_IP_address>:8088/
* Spark master: http://<dockerhadoop_IP_address>:8080/
* Spark worker: http://<dockerhadoop_IP_address>:8081/
* Hive: http://<dockerhadoop_IP_address>:10000

## Important note regarding Docker Desktop
Since Docker Desktop turned “Expose daemon on tcp://localhost:2375 without TLS” off by default there have been all kinds of connection problems running the complete docker-compose. Turning this option on again (Settings > General > Expose daemon on tcp://localhost:2375 without TLS) makes it all work. I’m still looking for a more secure solution to this.


## Quick Start HDFS

Copy Crimes_data.csv to the namenode.
```
  docker cp Crimes_data.csv namenode:Crimes_data.csv
```

Go to the bash shell on the namenode with that same Container ID of the namenode.
```
  docker exec -it namenode bash
```


Create a HDFS directory /data//openbeer/Crimes_data.

```
  hdfs dfs -mkdir -p /data/openbeer/Crimes_data
```

Copy breweries.csv to HDFS:
```
  hdfs dfs -put Crimes_data.csv /data/openbeer/Crimes_data/Crimes_data.csv
```


## Quick Start Spark (PySpark)

Go to http://<dockerhadoop_IP_address>:8080 or http://localhost:8080/ on your Docker host (laptop) to see the status of the Spark master.

Go to the command line of the Spark master and start PySpark.
```
  docker exec -it spark-master bash

  /spark/bin/pyspark --master spark://spark-master:7077
```

Load breweries.csv from HDFS.
```
  crimefile = spark.read.csv("hdfs://namenode:9000/data/openbeer/Crimes_data/Crimes_data.csv")
  
  crimefile.show()

```


## Quick Start Hive

Go to the command line of the Hive server and start hiveserver2

```
  docker exec -it hive-server bash

  hiveserver2
```

Maybe a little check that something is listening on port 10000 now
```
  netstat -anp | grep 10000
tcp        0      0 0.0.0.0:10000           0.0.0.0:*               LISTEN      446/java

```

Okay. Beeline is the command line interface with Hive. Let's connect to hiveserver2 now.

```
  beeline -u jdbc:hive2://localhost:10000 -n root
  
  !connect jdbc:hive2://127.0.0.1:10000 scott tiger
```

Didn't expect to encounter scott/tiger again after my Oracle days. But there you have it. Definitely not a good idea to keep that user on production.

Not a lot of databases here yet.
```
  show databases;
  
+----------------+
| database_name  |
+----------------+
| default        |
+----------------+
1 row selected (0.335 seconds)
```

Let's change that.

```
  create database openbeer;
  use openbeer;
```

And let's create a table.

```
CREATE EXTERNAL TABLE Chicago_Crimes_Data (
  ID STRING,
  `Case Number` STRING,
  `Date` STRING,
  Block STRING,
  IUCR STRING,
  `Primary Type` STRING,
  Description STRING,
  `Location Description` STRING,
  Arrest BOOLEAN,
  Domestic BOOLEAN,
  Beat INT,
  District INT,
  Ward STRING,
  `Community Area` INT,
  `FBI Code` STRING,
  `X Coordinate` INT,
  `Y Coordinate` INT,
  Year INT,
  `Updated On` STRING,
  Latitude DOUBLE,
  Longitude DOUBLE,
  Location STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION '/user/hive/warehouse/Crimes_Data.csv';
```


There you go: your private Hive server to play with.


## Configure Environment Variables

The configuration parameters can be specified in the hadoop.env file or as environmental variables for specific services (e.g. namenode, datanode etc.):
```
  CORE_CONF_fs_defaultFS=hdfs://namenode:8020
```

CORE_CONF corresponds to core-site.xml. fs_defaultFS=hdfs://namenode:8020 will be transformed into:
```
  <property><name>fs.defaultFS</name><value>hdfs://namenode:8020</value></property>
```
To define dash inside a configuration parameter, use triple underscore, such as YARN_CONF_yarn_log___aggregation___enable=true (yarn-site.xml):
```
  <property><name>yarn.log-aggregation-enable</name><value>true</value></property>
```

The available configurations are:
* /etc/hadoop/core-site.xml CORE_CONF
* /etc/hadoop/hdfs-site.xml HDFS_CONF
* /etc/hadoop/yarn-site.xml YARN_CONF
* /etc/hadoop/httpfs-site.xml HTTPFS_CONF
* /etc/hadoop/kms-site.xml KMS_CONF
* /etc/hadoop/mapred-site.xml  MAPRED_CONF

If you need to extend some other configuration file, refer to base/entrypoint.sh bash script.
