---
title: Setting up an Airflow Cluster
date: 2019-12-19 21:13:00 +0530
description: Airflow Cluster setup with Celery as an executor and PostgresSQL as the backend database.
categories:
- Airflow
tags:
- airflow
- postgresql
- data-pipeline
- aws
- rabbitmq
image: "/assets/airflow-image.jpeg"

---
Data-driven companies often hinge their business intelligence and product development on the execution of complex data pipelines. These pipelines are often referred to as data workflows, a term that can be somewhat opaque in that workflows are not limited to one specific definition and do not perform a specific set of functions per se. To orchestrate these workflows there are lot of schedulers like oozie, Luigi, Azkaban and Airflow. This blog demonstrate the setup of one of these orchestrator i.e Airflow. 

## A shot intro:

There are many orchestrators which are there in the technology space but Airflow provides a slight edge if our requirement hinges on of the following:

* No cron – With Airflows included scheduler we don't need to rely on cron to schedule our DAG and only use one framework (not like Luigi)
* Code Bases – In Airflow all the workflows, dependencies, and scheduling are done in python code. Therefore, it is rather easy to build complex structures and extend the flows.
* Language – Python is a language somewhat natural to pick up and was available on our team.

However setting up a production grade setup required some effort and this blog address the same.

## Basic Tech Terms:

* **Metastore:** Its a database which stores information regarding the state of tasks. Database updates are performed using an abstraction layer implemented in SQLAlchemy. This abstraction layer cleanly separates the function of the remaining components of Airflow from the database. 
* **Executor**: The Executor is a message queuing process that is tightly bound to the Scheduler and determines the worker processes that actually execute each scheduled task. 
* **Scheduler:** The Scheduler is a process that uses DAG definitions in conjunction with the state of tasks in the metadata database to decide which tasks need to be executed, as well as their execution priority. The Scheduler is generally run as a service.
* **Worker:** These are the processes that actually execute the logic of tasks, and are determined by the Executor being used.

## AWS Architecture: 

![](/assets/airflow-schematic.jpg)

Airflow provides an option to utilize CeleryExecutor to execute tasks in distributed fashion. In this mode, we can run several servers each running multiple worker nodes to execute the tasks. This mode uses Celery along with a message queueing service RabbitMQ.The diagram show the interactivity between different component services i.e. Airflow(Webserver and Scheduler), Celery(Executor) and RabbitMQ and Metastore in an AWS Environment. 

For simplicity of the blog, we will demonstrate the setup of a single node master server and a single node worker server. Below is the following details for the setup: 

* EC2 Master Node - Running Scheduler and Webserver
* EC2 Worker Node - Running Celery Executor and Workers
* RDS Metastore - Storing information about metadata and dag
* EC2 Rabbit MQ Nodes - Running RabbitMq broker

## Enviornment Prerequisite:

* Operating System: Ubuntu 16.04/Ubuntu 18.04 / Debian System
* Python Environment: Python 3.5x
* DataBase: PostgreSql v11.2 (RDS)

Once the prerequisites are taken care of, we can proceed with the installation.

## Installation:

The first step of the setup is the to configure the RDS Postgres database for airflow.
For that we need to connect to RDS Database using using admin user.For the sake of simplicity we are using command line utility from one of the EC2 servers to connect to our RDS Server. For command line client installation for postgres database on debian system execute following commands to install and execute.
 {% highlight shell %}
-- Client Installation
apt-get -y update 
apt-get install postgresql-client
{% endhighlight %}

Once the client is installed try to connect to the Database using the admin user.
 {% highlight shell %}
-- Generating IAM Token
export RDSHOST="{host_name}"
export PGPASSWORD="$(aws rds generate-db-auth-token --hostname $RDSHOST --port 5432 --region{region} --username {admin_user} )"

-- Connecting to the Database
psql "host=hostName port=portNumber dbname=DBName user=userName -password"
{% endhighlight %}

sslmode and sslrootcert parameter is used when we are using SSL/TLS based connection. For more information refere [here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.AWSCLI.PostgreSQL.html).

Once the connection is established with the database create a database named airflow which will act as a primary source where all the metadata,scheduler and other information will be stored by airflow.

 {% highlight shell %}
CREATE DATABASE airflow;

CREATE USER {DATABASE_USER} WITH PASSWORD ‘{DATABASE_USER_PASSWORD}’;
GRANT ALL PRIVILEGES ON DATABASE airflow TO {DATABASE_USER};
GRANT CONNECT ON DATABASE airflow TO {DATABASE_USER};
{% endhighlight %}


Once the above step is done the next step is to setup rabbitMQ in one the EC2 server. To install it follow the steps defined below.

* Login as root
* Install RabbitMQ Server
{ %highlight shell %}
apt-get install rabbitmq-server
{% endhighlight %}
* Verify status
{ %highlight shell %}
rabbitmqctl status
{% endhighlight %}
* Install RabbitMQ Web Interface
{ %highlight shell %}
rabbitmq-plugins enable rabbitmq_management
{% endhighlight %}


If you want to setup multiple machines to work as a RabbitMQ Cluster you can follow these instructions. Otherwise you can follow “Running RabbitMQ as Single Node” instructions to get it running on a single machine.

Note: The machines you want to use in the cluster need to be able to communicate with each other.

Follow the above “Install RabbitMQ” steps for on each node you want to add to the RabbitMQ Cluster
Ensure the RabbitMQ daemons are not running
See the “Running RabbitMQ as Single Node” section bellow
Choose one of the nodes as MASTER
On the non-MASTER nodes backup the .erlang.cookie file
mv /var/lib/rabbitmq/.erlang.cookie /var/lib/rabbitmq/.erlang.cookie.backup
Copy the file “/var/lib/rabbitmq/.erlang.cookie” from the MASTER node to the other nodes and store it at the same location.
Be careful during this step. If you copy the contents of the .erlang.cookie file and use the vi or nano editor to update the non-MASTER nodes .erlang.cookie file you may add a next line character to the file. You’re better off using an FTP service to copy the .erlang.cookie file down from the MASTER node and copy it onto the non-MASTER machines.
Set permissions of the .erlang.cookie file
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod 600 /var/lib/rabbitmq/.erlang.cookie
Startup the MASTER RabbitMQ Daemon in detached mode (as root)
rabbitmq-server -detached
On each of the of the non-MASTER nodes, add them to the cluster and start them up one at a time (as root)
#Stop App
rabbitmqctl stop_app
 
#Add the current machine to the cluster
rabbitmqctl join_cluster rabbit@{MASTER_HOSTNAME}
 
#Startup
rabbitmqctl start_app
 
#Check Status
rabbitmqctl cluster_status
The “Check Status” command should return something like the following:
Cluster status of node rabbit@{NODE_HOSTNAME} ...
[{nodes,[{disc,[rabbit@{MASTER_HOSTNAME},rabbit@{NODE_HOSTNAME}]}]},
 {running_nodes,[rabbit@{MASTER_HOSTNAME},rabbit@{NODE_HOSTNAME}]}]
Setup HA/Replication between Nodes
Access the Management URL of one of the nodes (See “Managing the RabbitMQ Instance(s)” section bellow)
Click on the Admin tab on top
Click on the Policies tab on the right
Add an HA policy
Name: ha-all
Pattern:
leave blank
Definitions:
ha-mode: all
ha-sync-mode: automatic
Priority: 0
Verify its setup correctly by navigating to the Queues section and clicking on one of the queues. You should see an entry in the Node and Slave section.
Setup a load balancer to balance requests between the the Nodes
Port Forwarding
Port 5672 (TCP) → Port 5672 (TCP)
Port 15672 (HTTP) → Port 15672 (HTTP)
Health Check
Protocol: HTTP
Ping Port: 15672
Ping Path: /
Point all processes to that LB

## Configuration:

We need to configure Zookeeper and Kafaka properties, Edit the `/etc/kafka/zookeeper.properties` on all the kafka nodes
 {% highlight shell %}
-- On Node 1
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=0
server.1=0.0.0.0:2888:3888
server.2=172.31.38.158:2888:3888
server.3=172.31.46.207:2888:3888
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
initLimit=5
syncLimit=2

-- On Node 2
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=0
server.1=172.31.47.152:2888:3888
server.2=0.0.0.0:2888:3888
server.3=172.31.46.207:2888:3888
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
initLimit=5
syncLimit=2

-- On Node 3
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=0
server.1=172.31.47.152:2888:3888
server.2=172.31.38.158:2888:3888
server.3=0.0.0.0:2888:3888
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
initLimit=5
syncLimit=2
{% endhighlight %}

We need to assign a unique ID for all the Zookeeper nodes. 
 {% highlight shell %}
 -- On Node 1
 echo "1" > /var/lib/zookeeper/myid
 
 --On Node 2
 echo "2" > /var/lib/zookeeper/myid
 
 --On Node 3
 echo "3" > /var/lib/zookeeper/myid
{% endhighlight %}

Now we need to configure Kafka broker. So edit the `/etc/kafka/server.properties` on all the kafka nodes.
 {% highlight shell %}
--On Node 1
broker.id.generation.enable=true
delete.topic.enable=true
listeners=PLAINTEXT://:9092
zookeeper.connect=172.31.47.152:2181,172.31.38.158:2181,172.31.46.207:2181
log.dirs=/kafkadata/kafka
log.retention.hours=168
num.partitions=1

--On Node 2
broker.id.generation.enable=true
delete.topic.enable=true
listeners=PLAINTEXT://:9092
log.dirs=/kafkadata/kafka
zookeeper.connect=172.31.47.152:2181,172.31.38.158:2181,172.31.46.207:2181
log.retention.hours=168
num.partitions=1

-- On Node 3
broker.id.generation.enable=true
delete.topic.enable=true
listeners=PLAINTEXT://:9092
log.dirs=/kafkadata/kafka
zookeeper.connect=172.31.47.152:2181,172.31.38.158:2181,172.31.46.207:2181
num.partitions=1
log.retention.hours=168
{% endhighlight %}

The next step is optimizing the `Java JVM Heap` size, In many places kafka will go down due to the less heap size. So Im allocating 50% of the Memory to Heap. But make sure more Heap size also bad. Please refer some documentation to set this value for very heavy systems.

 {% highlight shell %}
vi /usr/bin/kafka-server-start
export KAFKA_HEAP_OPTS="-Xmx2G -Xms2G"
{% endhighlight %}

The another major problem in the kafka system is the open file descriptors. So we need to allow the kafka to open at least up to 100000 files.
 {% highlight shell %}
vi /etc/pam.d/common-session
session required pam_limits.so

vi /etc/security/limits.conf

*                       soft    nofile          10000
*                       hard    nofile          100000
cp-kafka                soft    nofile          10000
cp-kafka                hard    nofile          100000
{% endhighlight %}

Here the `cp-kafka` is the default user for the kafka process.

### Create Kafka data dir:
 {% highlight shell %}
mkdir -p /kafkadata/kafka
chown -R cp-kafka:confluent /kafkadata/kafka
chmode 710 /kafkadata/kafka
{% endhighlight %}

### Start the Kafka cluster:
 {% highlight shell %}
sudo systemctl start confluent-zookeeper
sudo systemctl start confluent-kafka
sudo systemctl start confluent-schema-registry
{% endhighlight %}

Make sure the Kafka has to automatically starts after the Ec2 restart.
 {% highlight shell %}
sudo systemctl enable confluent-zookeeper
sudo systemctl enable confluent-kafka
sudo systemctl enable confluent-schema-registry
{% endhighlight %}

Now our kafka cluster is ready. To check the list of system topics run the following command.
 {% highlight shell %}
kafka-topics --list --zookeeper localhost:2181

__confluent.support.metrics
{% endhighlight %}

## Setup Debezium:

Install the confluent connector and debezium MySQL connector on all the producer nodes.
 {% highlight shell %}
apt-get update 
sudo apt-get install default-jre
 
wget -qO - https://packages.confluent.io/deb/5.3/archive.key | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/5.3 stable main"
sudo apt-get update && sudo apt-get install confluent-hub-client confluent-common confluent-kafka-connect-s3 confluent-kafka-2.12
{% endhighlight %}

### Configuration:

Edit the `/etc/kafka/connect-distributed.properties` on all the producer nodes to make our producer will run on a distributed manner.
 {% highlight shell %}
-- On all the connector nodes
bootstrap.servers=172.31.47.152:9092,172.31.38.158:9092,172.31.46.207:9092
group.id=debezium-cluster
plugin.path=/usr/share/java,/usr/share/confluent-hub-components
{% endhighlight %}

### Install Debezium MySQL Connector:

 {% highlight shell %}
confluent-hub install debezium/debezium-connector-mysql:latest
{% endhighlight %}

it'll ask for making some changes just select `Y` for everything.

### Run the distributed connector as a service:
 {% highlight shell %}
vi /lib/systemd/system/confluent-connect-distributed.service

[Unit]
Description=Apache Kafka - connect-distributed
Documentation=http://docs.confluent.io/
After=network.target

[Service]
Type=simple
User=cp-kafka
Group=confluent
ExecStart=/usr/bin/connect-distributed /etc/kafka/connect-distributed.properties
TimeoutStopSec=180
Restart=no

[Install]
WantedBy=multi-user.target
{% endhighlight %}

### Start the Service:
 {% highlight shell %}
systemctl enable confluent-connect-distributed
systemctl start confluent-connect-distributed
{% endhighlight %}

## Configure Debezium MySQL Connector:

Create a `mysql.json` file which contains the MySQL information and other formatting options.
 {% highlight json %}
{
	"name": "mysql-connector-db01",
	"config": {
		"name": "mysql-connector-db01",
		"connector.class": "io.debezium.connector.mysql.MySqlConnector",
		"database.server.id": "1",
		"tasks.max": "3",
		"database.history.kafka.bootstrap.servers": "172.31.47.152:9092,172.31.38.158:9092,172.31.46.207:9092",
		"database.history.kafka.topic": "schema-changes.mysql",
		"database.server.name": "mysql-db01",
		"database.hostname": "172.31.84.129",
		"database.port": "3306",
		"database.user": "bhuvi",
		"database.password": "my_stong_password",
		"database.whitelist": "proddb,test",
		"internal.key.converter.schemas.enable": "false",
		"key.converter.schemas.enable": "false",
		"internal.key.converter": "org.apache.kafka.connect.json.JsonConverter",
		"internal.value.converter.schemas.enable": "false",
		"value.converter.schemas.enable": "false",
		"internal.value.converter": "org.apache.kafka.connect.json.JsonConverter",
		"value.converter": "org.apache.kafka.connect.json.JsonConverter",
		"key.converter": "org.apache.kafka.connect.json.JsonConverter",
		"transforms": "unwrap",
		"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.add.source.fields": "ts_ms",
		"tombstones.on.delete": false
	}
}
{% endhighlight %}

* "database.history.kafka.bootstrap.servers" - Kafka Servers IP.
* "database.whitelist" - List of databases to get the CDC.
* key.converter and value.converter and transforms parameters - By default Debezium output will have more detailed information. But I don't want all of those information. Im only interested in to get the new row and the timestamp when its inserted.

If you don't want to customize anythings then just remove everything after the `database.whitelist`

### Register the MySQL Connector:
 {% highlight shell %}
curl -X POST -H "Accept: application/json" -H "Content-Type: application/json" http://localhost:8083/connectors -d @mysql.json
{% endhighlight %}

### Check the status:
 {% highlight shell %}
curl GET localhost:8083/connectors/mysql-connector-db01/status
{
  "name": "mysql-connector-db01",
  "connector": {
    "state": "RUNNING",
    "worker_id": "172.31.94.191:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "172.31.94.191:8083"
    }
  ],
  "type": "source"
}
{% endhighlight %}

### Test the MySQL Consumer: 

Now insert something into any tables in `proddb or test` (because we have whilelisted only these databaes to capture the CDC. 
 {% highlight sql %}
use test;
create table rohi (id int,
fn varchar(10),
ln varchar(10),
phone int );

insert into rohi values (2, 'rohit', 'ayare','87611');
{% endhighlight %}

We can get these values from the Kafker brokers. Open any one the kafka node and run the below command.

I prefer confluent cli for this. By default it'll not be available, so download manually.
 {% highlight shell %}
curl -L https://cnfl.io/cli | sh -s -- -b /usr/bin/
{% endhighlight %}

### Listen the below topic:

> **mysql-db01.test.rohi**   
> This is the combination of `servername.databasename.tablename`   
> servername(you mentioned this in as a server name in mysql json file).
 {% highlight shell %}
confluent local consume mysql-db01.test.rohi

----
The local commands are intended for a single-node development environment
only, NOT for production usage. https://docs.confluent.io/current/cli/index.html
-----

{"id":1,"fn":"rohit","ln":"ayare","phone":87611,"__ts_ms":1576757407000}
{% endhighlight %}

## Setup S3 Sink connector in All Producer Nodes:

I want to send this data to S3 bucket. So you must have an EC2 IAM role which has access to the target S3 bucket. Or install `awscli` and configure access and secret key(but its not recommended)

### Install S3 Connector:
{% highlight shell %}
confluent-hub install confluentinc/kafka-connect-s3:latest
{% endhighlight %}


Create `s3.json` file.
 {% highlight json %}
{
	"name": "s3-sink-db01",
	"config": {
		"connector.class": "io.confluent.connect.s3.S3SinkConnector",
		"storage.class": "io.confluent.connect.s3.storage.S3Storage",
		"s3.bucket.name": "bhuvi-datalake",
		"name": "s3-sink-db01",
		"tasks.max": "3",
		"s3.region": "us-east-1",
		"s3.part.size": "5242880",
		"s3.compression.type": "gzip",
		"timezone": "UTC",
		"locale": "en",
		"flush.size": "10000",
		"rotate.interval.ms": "3600000",
		"topics.regex": "mysql-db01.(.*)",
		"internal.key.converter.schemas.enable": "false",
		"key.converter.schemas.enable": "false",
		"internal.key.converter": "org.apache.kafka.connect.json.JsonConverter",
		"format.class": "io.confluent.connect.s3.format.json.JsonFormat",
		"internal.value.converter.schemas.enable": "false",
		"value.converter.schemas.enable": "false",
		"internal.value.converter": "org.apache.kafka.connect.json.JsonConverter",
		"value.converter": "org.apache.kafka.connect.json.JsonConverter",
		"key.converter": "org.apache.kafka.connect.json.JsonConverter",
		"partitioner.class": "io.confluent.connect.storage.partitioner.HourlyPartitioner",
		"path.format": "YYYY/MM/dd/HH",
		"partition.duration.ms": "3600000",
		"rotate.schedule.interval.ms": "3600000"
	}
}
{% endhighlight %}

* `"topics.regex": "mysql-db01"` - It'll send the data only from the topics which has `mysql-db01` as prefix. In our case all the MySQL databases related topics will start  with this prefix.
* `"flush.size"` - The data will uploaded to S3 only after these many number of records stored. Or after `"rotate.schedule.interval.ms"` this duration.

### Register this S3 sink connector:
{% highlight shell %}
curl -X POST -H "Accept: application/json" -H "Content-Type: application/json" http://localhost:8083/connectors -d @s3
{% endhighlight %}

### Check the Status: 
 {% highlight shell %}
curl GET localhost:8083/connectors/s3-sink-db01/status
{
  "name": "s3-sink-db01",
  "connector": {
    "state": "RUNNING",
    "worker_id": "172.31.94.191:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "172.31.94.191:8083"
    },
    {
      "id": 1,
      "state": "RUNNING",
      "worker_id": "172.31.94.191:8083"
    },
    {
      "id": 2,
      "state": "RUNNING",
      "worker_id": "172.31.94.191:8083"
    }
  ],
  "type": "sink"
}
{% endhighlight %}
### Test the S3 sync:

Insert the 10000 rows into the `rohi` table. Then check the S3 bucket. It'll save the data in JSON format with GZIP compression. Also in a HOUR wise partitions. 

![](/assets/Build Production Grade Dedezium Cluster With Confluent Kafka-1.jpg)

![](/assets/Build Production Grade Dedezium Cluster With Confluent Kafka-2.jpg)

## Monitoring:
Refer [this post](https://thedataguy.in/monitor-debezium-mysql-connector-with-prometheus-and-grafana/) to setup monitoring for MySQL Connector.

## More Tuning: 

* Replication Factor is the other main parameter to the data durability. 
* Use internal IP addresses as much as you can. 
* By default debezium uses 1 Partition per topic. You can configure this based on your work load. But more partitions more through put needed. 

## References: 

1. [Setup Kafka in production by confluent](https://docs.confluent.io/current/kafka/deployment.html)
2. [How to choose number of partition](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/)
3. [Open file descriptors for Kafka ](https://log-it.ro/2017/10/16/ubuntu-change-ulimit-kafka-not-ignore/)
4. [Kafka best practices in AWS](https://aws.amazon.com/blogs/big-data/best-practices-for-running-apache-kafka-on-aws/)
5. [Debezium documentation](https://debezium.io/documentation/reference/1.0/tutorial.html)
6. [Customize debezium output with SMT](https://debezium.io/documentation/reference/1.0/configuration/event-flattening.html)

### Debezium Series blogs:

1. [Build Production Grade Debezium Cluster With Confluent Kafka](https://thedataguy.in/build-production-grade-debezium-with-confluent-kafka-cluster/)
2. [Monitor Debezium MySQL Connector With Prometheus And Grafana](https://thedataguy.in/monitor-debezium-mysql-connector-with-prometheus-and-grafana/)
3. [Debezium MySQL Snapshot From Read Replica With GTID](https://thedataguy.in/debezium-mysql-snapshot-from-read-replica-with-gtid/)
4. [Debezium MySQL Snapshot From Read Replica And Resume From Master](https://thedataguy.in/debezium-mysql-snapshot-from-read-replica-and-resume-from-master/)
5. [Debezium MySQL Snapshot For AWS RDS Aurora From Backup Snaphot](https://thedataguy.in/debezium-mysql-snapshot-for-aws-rds-aurora-from-backup-snaphot/)
6. [RealTime CDC From MySQL Using AWS MSK With Debezium](https://medium.com/searce/realtime-cdc-from-mysql-using-aws-msk-with-debezium-28da5a4ca873)