# How to use Percona-pxc-mysql with DC/OS

## Prerequisites

* A running DC/OS 1.11 cluster

## Install

Before starting with installation of Percona-pxc-mysql, we need to install the kafka-zookeeper service on the same DC/OS cluster. Please refer kafka-zookeeper service docs as installation guide.

For a basic kafka-zookeeper install from command line we need to use the following command:

```bash
dcos package install kafka-zookeeper
```

and via DC/OS UI, kafka-zookeeper can be installed as shown below:

![Kafka-Zookeeper Install](img/kafka-zookeeper-ui-install.png)

Percona-pxc-mysql can be installed via either the DC/OS Catalog UI or by using the CLI. The following command will launch the install via the DC/OS CLI:

```bash
dcos package install percona-pxc-mysql
```

![PXC Install](img/pxc_install.png)

In either case a default cluster will come up with three Percona-pxc-mysql nodes and one ProxySQL node on the public DC/OS cluster node.

Proxysql is the proxy which has will be deployed on the public agent of DC/OS cluster.
It serves as a proxy/bridge between external mysql clients and the Percona-pxc-mysql cluster which resides on the private agents of the DC/OS cluster.
Proxysql has been configured to use pod 0 of Percona-pxc-mysql as the read and write node and other nodes have been configured as reader nodes as a part of default install. 

So one can connect from an external mysql 5.7 client to the Percona-pxc-mysql cluster via ProxySQL.

It is recommended to have nodes in odd number count for Percona-pxc-mysql to avoid condition of split brain.


## Accessing the Console

Once the cluster is up and running, we need to have a mysql client version 5.7 on our local system to connect to the pxc cluster.

From a linux terminal, with docker installed, we can use the following command to spin up a mysql 5.7 client container

```bash
docker run --name mysql-cli -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

Next we need to run the following command to attach to the running container 


```bash
docker exec -ti mysql-cli bash
```

Doing this we will be inside the mysql client container and we can verify the mysql version by the following command 

```bash
mysql --version
mysql  Ver 14.14 Distrib 5.7.22, for Linux (x86_64) using  EditLine wrapper
```

Now we need to find out Public facing DC/OS cluster hostname/IP for connecting from the mysql client.

In our case since our DC/OS cluster is deployed on AWS infrastructure, we need to get the EC2 instance in the DC/OS cluster which is part of the "PublicSlaveSecurityGroup". In our case it is 13.114.101.88

Hence we us the following command using mysql client to connect to the Percona-pxc-mysql cluster on DC/OS :

```bash
root@bf7132b3a085:/# mysql -uproxymon -pproxym0n -h 13.114.101.88 -P 3306
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.5.30 (ProxySQL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```

Now we will check some basic commands to see whether we are able to reach the Percona-pxc-mysql cluster :

```bash

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.12 sec)


mysql> select 1;
+---+
| 1 |
+---+
| 1 |
+---+
1 row in set (0.14 sec)

```

As a next step we will create a table in the Percona-pxc-mysql cluster using the following steps:

1. Create a database schema

```bash
mysql> create database mydb;
Query OK, 1 row affected (0.12 sec)

```
2. Switch to above schema

```bash
mysql> use mydb;
Database changed
```

3. Create a table

```bash
mysql> CREATE TABLE Persons (PersonID int, LastName varchar(255), FirstName varchar(255), Address varchar(255), City varchar(255), PRIMARY KEY(PersonID));
Query OK, 0 rows affected (0.21 sec)
```

4. Insert a record into the above table and commit

```bash
mysql> insert into Persons values ('1', 'Tom', 'Erichsen', 'Skagen 21', 'Stava');
Query OK, 1 row affected (0.15 sec)

mysql> commit;
Query OK, 0 rows affected (0.12 sec)
```
5. check the records in the above table

```bash
mysql> select * from Persons;
+----------+----------+-----------+-----------+-------+
| PersonID | LastName | FirstName | Address   | City  |
+----------+----------+-----------+-----------+-------+
|        1 | Tom      | Erichsen  | Skagen 21 | Stava |
+----------+----------+-----------+-----------+-------+
1 row in set (0.11 sec)
```


Now we need to add more query rules in the proxysql service by connecting to it by referring to https://github.com/sysown/proxysql/wiki/ProxySQL-Configuration.  

We will also need to get the proxysql endpoint hostname for connecting to it from the task using the below command:

```bash
dcos percona-pxc-mysql endpoints lb-port
```

To connect to the proxysql service running on the cluster, we need to get the  task id for proxysql service using the below command.

```bash
$ dcos task
```
Then we need to get into the proxysql task by assigning bash shell to it in the following way

```bash
dcos task exec -ti proxysql-0-proxysql-task__cc6eb6cd-054e-460f-9a9b-842bda3930a4 bash
```

From the proxysql task we need to connect to the proxysql service using the below command:

```bash
mysql -uradminuser -pradminpass -h proxysql-0-proxysql-task.pxc.autoip.dcos.thisdcos.directory -P 6032
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 82334
Server version: 5.5.30 (ProxySQL Admin Module)

Copyright (c) 2009-2018 Percona LLC and/or its affiliates
Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>     
```
Now we can setup query rules following the above mentioned link for proxysql configuration.



## Adding Nodes

Let’s say we have 3 Percona-pxc-mysql nodes, and we want to go to three.

![PXC Nodes](img/PXC_Nodes.png)

You need to go and edit the configuration of your couchbase service, and increase the Percona-pxc-mysql node count to 5.

![PXC Node add configuration](img/PXC_add_nodes.png)


## Limitations

Limitations of the framework:

* Support for fault domains in the form of zones
* Percona-pxc-mysql and Proxysql related configuration parameters are currently not exposed. 
* TLS for proxysql
* Backup and restore
* Non-root install and install on Strict mode DC/OS cluster
* Encryption for data at rest
* MOUNT volumes setup is not tested
* Emit app metrics to DC/OS metrics API
* Proxysql implementation with zookeeper based service discovery 
