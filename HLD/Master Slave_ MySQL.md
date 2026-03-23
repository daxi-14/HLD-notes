

# **Scaling Databases: Master Slave**

# Script

* Present a hypothetical case study on why a single node deployment is a bottleneck and why we need master-slave replication  
  * Increased read throughput, load balance read queries  
  * High availability and instant failover with no data loss  
  * Backups and data distribution  
* Discuss how replication works on MySQL  
  * binlog, relay log, IO threads  
* Setup a simple master-slave without any data  
  * using Docker  
  * using AWS  
  * perform some DDL and DML operations and show that the replica picks up the changes  
  * Show how to add existing data  
* Sync vs Async Replication  
  * Show AWS Multi-AZ deployment  
* Statement-based vs Row-based replication  
  * Differences  
  * Examples

Extra things (not added right now) we can cover for advanced folks

* GTID  
* Replication lag  
  * Why replication lag happens,   
  * Reading your Writes  
* Load balancing read queries using master \+ 2 slave nodes  
  * HAProxy  
  * ProxySQL  
* Custom implementation of binlog replication: MySQL master, Postgres slave \-\> Data pipeline/warehousing example  
  * using Debezium \+ Kafka \+ Change Data Capture

Disclaimer: This tutorial assumes some familiarity with Docker and any application framework (like Django/ Spring/ Ruby on Rails/ Laravel) using a SQL database (MySQL). Check the Appendix for guides on how to set up Docker on your machine.

# Introduction: Story

We are TunTun Ladoo, a leading online retailer and one-stop shop for ladoos. Like every year, this Diwali season we're planning a massive Dhamaka sale, where hundreds of thousands of visitors will shop ladoos on our website and app in a matter of hours. 

Last year due to our growing popularity we went down during peak hours. Our production MySQL DB wasn't able to handle the load and took a downtime of about an hour. It was bad, we lost a bunch of revenue and in the worst case, the database could have been corrupted, taking our site down for days.

In the postmortem, we discovered that we have a bunch of expensive SQL queries which increases the overall CPU and Memory pressure of our MySQL instance. The majority of these queries were read-only, joining data across multiple large tables. For example, queries to check the stock, pricing, and inventory of Ladoos,  analytical queries for checking sales and orders in real-time.

This year we're gearing up not to repeat the same, we have a plan to scale our MySQL. Our goals are pretty straightforward:

* Our read load during peak sales is higher than what a single machine can handle. We need to fix this bottleneck. A simple approach would be to buy a more powerful machine i.e. vertical scaling. But it's costly and after a certain point, we won't be able to scale anymore. Instead, we can potentially distribute the load across machines, i.e scaling horizontally.  
* With multiple MySQL nodes, we get better redundancy. Even if one of these nodes goes down or crashes, another one can take over immediately.  
* Our OLTP MySQL database is not meant for processing large analytical queries containing multiple joins and aggregations. An OLAP database is better suited for this purpose. We need to capture the changes in our production database and sync them to our data warehouse in real-time.

Let's assume that our dataset is static, i.e. it doesn't change over time. Scaling our reads is super easy in that case. We just have to copy the data to each node once and we are good to go. But in the real world, as our tables are constantly changing, we need to handle the *changes* in replicated data and propagate them to each replica.

# Replication

Luckily for us, MySQL's built-in replication solves the problem of keeping one server's data synchronized with another's. In this case, we need to mark one of the servers as a leader and MySQL will keep other nodes data in sync with the master copy.

## More On Master Slave

In this setup, we would have multiple MySQL nodes, one of which would be termed the leader/master/ primary. *All writes from clients need to be processed and persisted by the leader.* Other nodes are called followers/ replica/ slaves and are read-only from the client's viewpoint.  
After persisting the data locally, the leader node sends the data change to followers using the *replication log/change stream*. Each follower takes the log and updates its local copy accordingly, making sure writes are applied in the same order as processed on the leader.  
When reading data, clients can query either leader or any of the followers, however, writes are only processed by the leader.

# Setting up Replication on MySQL

At a higher level, MySQL replication consists of three different parts. Before committing every transaction that updates data, the master node records the changes in the binary log file in a serialized way (even if the transactions were running concurrently). During boot up, replica nodes start a worker thread called *IO slave thread* which maintains a connection to the master node and starts the *binlog dump* process. This binlog dump process reads data from the master's binary log and dumps it in its local *relay log*. It waits for signals from the master node for any future updates. The SQL slave thread reads and replays events from the relay log, updating the replica's data to be in sync with the master. The process of fetching and replaying events is decoupled and runs asynchronously. But as the replica uses two different threads for fetching and processing events, replication needs to be serialized. Transactions that might have run in parallel on the master are processed serially on the replica.

Setting up replication for a blank server is fairly simple. In the master nodes configuration we need to enable binlog by setting log\_bin \= 1 and specify the logging format binlog\_format as ROW. For servers used in replication, we need to specify a unique server ID for every node.

*`# master.conf`*  
`[mysqld]`  
`server-id = 1`  
`log_bin = 1`  
`binlog_format = ROW`  
`binlog_do_db = scalerdb`  
`default_authentication_plugin = mysql_native_password`

*`# slave.conf`*  
`[mysqld]`  
`server-id = 2`  
`log_bin = 1`  
`binlog_do_db = scalerdb`  
`default_authentication_plugin = mysql_native_password`

`➜ docker run \`  
 `--name masternode --rm \`  
 `-e MYSQL_DATABASE=scalerdb \`  
 `-e MYSQL_USER=master_user \`  
 `-e MYSQL_PASSWORD=master_pwd \`  
 `-e MYSQL_ROOT_PASSWORD=111 \`  
 `-v $PWD/master.conf:/etc/mysql/conf.d/mysql.conf.cnf \`  
 `mysql:8`

`➜ docker run \`  
 `--name slavenode --rm \`  
 `-e MYSQL_DATABASE=scalerdb \`  
 `-e MYSQL_USER=slave_user \`  
 `-e MYSQL_PASSWORD=slave_pwd \`  
 `-e MYSQL_ROOT_PASSWORD=111 \`  
 `-v $PWD/slave.conf:/etc/mysql/conf.d/mysql.conf.cnf \`  
 `mysql:8`

The slavenode IO thread, which makes a TCP connection to masternode needs permissions to connect as a user and read the master's binary log file.  
*`# Create a replication user, SSH into master node and get a local`*   
*`# MySQL CLI`*  
`➜ docker exec -it masternode /bin/bash -c "mysql -u root --password=111"`

`mysql> CREATE USER "replication_user"@"%" IDENTIFIED BY "replication_password";`  
`Query OK, 0 rows affected (0.01 sec)`

`mysql> GRANT REPLICATION SLAVE ON *.* TO "replication_user"@"%";`  
`Query OK, 0 rows affected (0.00 sec)`

`mysql> FLUSH PRIVILEGES;`  
`Query OK, 0 rows affected (0.00 sec)`

`mysql> SHOW MASTER STATUS;`  
`+----------+----------+--------------+------------------+-------------------+`  
`| File     | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |`  
`+----------+----------+--------------+------------------+-------------------+`  
`| 1.000003 |      851 | scalerdb     |                  |                   |`  
`+----------+----------+--------------+------------------+-------------------+`  
`1 row in set (0.00 sec)`

*`# Get the masternode IP address and open slavenode CLI`*  
`➜ docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' masternode`  
`172.17.0.2`

`➜ docker exec -it slavenode /bin/bash -c "mysql -u root --password=111"`

`mysql> SHOW SLAVE STATUS;`  
*`# Empty set, 1 warning (0.00 sec)`*  
`mysql> CHANGE MASTER TO`  
   `->   MASTER_HOST='172.17.0.2',`  
   `->   MASTER_USER='replication_user',`  
   `->   MASTER_PASSWORD='replication_password',`  
   `->   MASTER_LOG_FILE='1.000003',`  
   `->   MASTER_LOG_POS=851;`  
*`# Query OK, 0 rows affected, 8 warnings (0.03 sec)`*  
`mysql> START SLAVE;`

`mysql> CREATE TABLE customers (`  
   `->     id int NOT NULL AUTO_INCREMENT,`  
   `->     firstname varchar(255),`  
   `->     lastname varchar(255),`  
   `->     address varchar(255),`  
   `->     city varchar(255),`  
   `->     unique_id varchar(255),`  
   `->     CONSTRAINT pk_customers PRIMARY KEY (id)`  
   `-> );`  
*`# Query OK, 0 rows affected (0.02 sec)`*

`mysql>`  
`mysql> INSERT INTO customers (firstname, lastname, address, city, unique_id)`  
   `-> VALUES ('Joydeep', 'Mukherjee', 'Sakti Statesman, Bellandur', 'Bangalore', UUID());`  
*`# Query OK, 1 row affected (0.01 sec)`*

`mysql> use scalerdb;`  
`# Database changed`  
`mysql>`  
`mysql> select * from customers;`  
`# Empty set (0.00 sec)`

`mysql> select * from customers;`  
`+----+-----------+-----------+----------------------------+-----------+-------------------------+`  
`| id | firstname | lastname  | address                    | city      | unique_id               |`  
`+----+-----------+-----------+----------------------------+-----------+-------------------------+`  
`|  1 | Joydeep   | Mukherjee | Sakti Statesman, Bellandur | Bangalore | 4f982cf4-2637-11ed-aec3 |`  
`+----+-----------+-----------+----------------------------+-----------+-------------------------+`  
`# 1 row in set (0.00 sec)`

### AWS Example

[How do I create a read replica for an Amazon RDS database?](https://www.youtube.com/watch?v=COsx7UrMGL4)

### Adding a Node Later on

1. Take a consistent snapshot of the leaders' database at some point, without locking the whole database, and associate the snapshot with an exact position in binlog.  
2. Copy the snapshot to a new replica and connect to the leader. It will request all the changes that have happened since the snapshot was taken (*binlog coordinates*)  
3. Eventually, the follower will process the backlog of changes.

# Replication Types

*TODO: Replace graphics*  
When updating the price of Ladoos, the master node will receive an update request from our application servers at some point. Once the master updates its local copy, different replication strategies can be used to broadcast the changes to replicas. Replication between the master and slaves can happen synchronously or asynchronously. 

## Synchronous Replication

In synchronous replication, the master node notifies all the replicas (sequentially/ parallelly) and waits until all the slaves have confirmed the write operation to go through, before reporting success to the user and making the write visible to other queries.

Synchronous replication ensures that the replicas are always in sync with the master node. Even in case of a master node crash, replicas will still have consistently updated data making the system fault-tolerant by design. We can quickly promote any one of the slave nodes as the new master and continue to function as usual.

However, if a synchronous slave does not respond either due to a crash or network partition, write operations won't be processed. The master node needs to block all write operations till the replica comes back online. 

Ideally, it's recommended to have only one slave as a synchronous replica. This guarantees that we have an up-to-date copy of data on at least two nodes.

## Asynchronous Replication

In asynchronous replication, the master completes the write operation immediately and responds back to the client without waiting for the change to be propagated to the replicas. In this case, the clients will not be blocked for a long duration, increasing the overall throughput. If the master node crashes and is not recoverable, any writes that have not been propagated are lost. This means a write is not guaranteed to be durable even if it has been confirmed to the client.  
![][image1]

### AWS Example

Enable the Multi-AZ option for RDS and perform a failover (IP change)

# 

# Replication Variants

## Statement based

Statement based replication works by recording queries that change data on the master. This means the leader logs every INSERT, UPDATE or DELETE statement in its binlog and sends them to replica. Replica instances reads the events from the relay log as actual SQL statements and reexecutes them as if it had been received from a client.

In theory, this will keep the replica in sync with the master and as we are using SQL statements, the binlog events tend to be compact and doesnt use a lot of bandwidth. (A query that updates gigabytes of data might add a few kb overhead in binlog). However there are several drawbacks:

* Any statement with a nondeterministic function like NOW(), RAND(), UUID(), is likely to generate different values on each node  
* Statements with side-effects like triggers, stored procedures may result in different side-effects  
* Statements depending on existing data in the database, must be executed in the same order on each replica. Thus all modifications must be serializable on replica, requiring more locking.

TODO: Add an example of a non-deterministic update using UUID() with statement based replication and show the inconsistency 

## Row-based

In case of row based replication, MySQL records the actual data changes in the binary log. This ensures every statement is replicated correctly. Row based replication can be extremely efficient and inefficient at the same time depending on the query.

`INSERT INTO analytics(total)`  
 `SELECT sum(price) FROM very_large_invoice_table;`

This query will scan many rows in the source table but will result in a single binlog event. In case of statement based replication, this query would have to be processed again on all the replica nodes.

`UPDATE very_large_invoice_table SET discount = 0;`

However, row based replication will be very expensive in this case as every row needs to be written to the binary log, making the binary log event extremely large. 

MySQL can switch between statement based and row based replication dynamically if we set binlog\_format \= MIXED

## Write Ahead Logs 

\# TODO

# Replication Lag

\#TODO

# Extra Examples

## Example 3: Read Load Balancing: Multi node replicas

[https://shashwotrisal.medium.com/load-balancing-mysql-servers-using-proxysql-ff5ab6d1d4ea](https://shashwotrisal.medium.com/load-balancing-mysql-servers-using-proxysql-ff5ab6d1d4ea)   
[https://towardsdatascience.com/high-availability-mysql-cluster-with-load-balancing-using-haproxy-and-heartbeat-40a16e134691](https://towardsdatascience.com/high-availability-mysql-cluster-with-load-balancing-using-haproxy-and-heartbeat-40a16e134691) 

## Example 4: MySQL master, Postgres Slave: OLTP to OLAP

[https://medium.com/event-driven-utopia/a-visual-introduction-to-debezium-32563e23c6b8](https://medium.com/event-driven-utopia/a-visual-introduction-to-debezium-32563e23c6b8)   
[https://medium.com/event-driven-utopia/configuring-debezium-to-capture-postgresql-changes-with-docker-compose-224742ca5372](https://medium.com/event-driven-utopia/configuring-debezium-to-capture-postgresql-changes-with-docker-compose-224742ca5372) 

# References

Kleppmann: Chapter 5

[https://scribe.rip/swlh/zero-downtime-master-slave-replication-4f2814138edf](https://scribe.rip/swlh/zero-downtime-master-slave-replication-4f2814138edf)   
[https://datto.engineering/post/lossless-mysql-semi-sync-replication-and-automated-failover](https://datto.engineering/post/lossless-mysql-semi-sync-replication-and-automated-failover)   
[https://severalnines.com/resources/whitepapers/mysql-replication-high-availability/](https://severalnines.com/resources/whitepapers/mysql-replication-high-availability/) 

[https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql](https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql)   
[Turns out MySQL Statement-based Replication might not be a good idea, Lets discuss why](https://www.youtube.com/watch?v=WSDjjgSP-jA)  
[https://engineering.fb.com/2021/07/22/data-infrastructure/mysql/](https://engineering.fb.com/2021/07/22/data-infrastructure/mysql/)   
[https://www.uber.com/en-IN/blog/postgres-to-mysql-migration/](https://www.uber.com/en-IN/blog/postgres-to-mysql-migration/)   
[https://arpitbhayani.me/blogs/replication-strategies](https://arpitbhayani.me/blogs/replication-strategies)   


[image1]: <data:image/png;base64,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>