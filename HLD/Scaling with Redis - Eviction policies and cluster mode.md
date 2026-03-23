# **Scaling with Redis \- Eviction policies and cluster mode**

Setting up Redis cluster, LRU with limited space, show how eviction happens as you push keys. Simple get, set operations. For more operations, we can redirect to [https://try.redis.io/](https://try.redis.io/) in the video.

## **Script**

* Motivation \- Why do we need Redis?  
  * Alternatives \- What are some other solutions to the problems presented?  
* Redis  
  * What is Redis?  
  * How does Redis work?  
  * Installation  
* Basic Redis commands  
  * SET  
  * GET  
  * DEL  
  * INCR  
  * EXISTS  
  * CONFIG  
* Eviction policies  
  * Background \- What is an eviction policy?  
  * Setup \- Using Redis with limited space  
  * No eviction \- What happens when the memory limit is reached?  
  * LRU \- Least Recently Used eviction policy  
  * Other eviction policies  
* Cluster mode  
  * Background \- What is cluster mode?  
  * Setup \- Setting up Redis cluster  
  * Operations \- Simple get, set operations  
  * 

## **Table of contents**

* [Scaling with Redis \- Eviction policies and cluster mode](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#scaling-with-redis---eviction-policies-and-cluster-mode)  
  * [Script](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#script)  
  * [Table of contents](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#table-of-contents)  
  * [Motivation](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#motivation)  
    * [Local cache](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#local-cache)  
    * [Global cache](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#global-cache)  
    * [Distributed cache](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#distributed-cache)  
  * [Redis](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#redis)  
    * [Usage](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#usage)  
      * [Setup](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#setup)  
      * [Commands](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#commands)  
      * [Keys](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#keys)  
  * [Eviction policies](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#eviction-policies)  
    * [LRU](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#lru)  
  * [Cluster mode](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#cluster-mode)  
    * [Motivation](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#motivation-1)  
    * [Specification](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#specification)  
      * [Sharding](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#sharding)  
      * [Replication](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#replication)  
      * [Configuration](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#configuration)  
    * [Setup](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#setup-1)  
    * [Testing the cluster](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#testing-the-cluster)  
  * [Code Glossary](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#code-glossary)  
  * [References](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ#references)

## **Motivation**

The new year is just around the corner which means it is time for some shopping and crazy sales. You are a developer at a popular e-commerce website Nile.com. Currently, you have a very simple setup where you have few servers behind a load balancer and a database server.

![][image1]

graph LR

    subgraph clients\[Clients\]  
        webapp\[Web App\]  
        mobile\[Mobile App\]  
        api\[API\]  
    end

    LB\[Load Balancer\]

    subgraph Backend Servers  
        S1\[Server 1\]  
        S2\[Server 2\]  
        S3\[Server 3\]  
    end

    DB\[(Database)\]

    webapp \--\> LB  
    mobile \--\> LB  
    api \--\> LB

    LB \--\> S1  
    LB \--\> S2  
    LB \--\> S3

    S1 \--\> DB  
    S2 \--\> DB  
    S3 \--\> DB

As the number of users increase, you start to see some performance issues. The database server is getting overloaded, and the application servers are not able to handle the load. The average response time for the APIs is increasing and is well above the acceptable threshold of 200ms.

You decide to do some digging and find out that the database is getting overloaded because of the number of queries being made. You see that there are a lot of duplicate queries being made. This is a prime candidate for caching.

### **Local cache**

You decide to add a local cache to the application servers which will store the results of the popular queries. This will reduce the number of queries being made to the database. The data can be stored in memory, which is fast, but the memory is limited. You can also store the data on disk, which is slower, but has more space. However, the access time in each case is still faster than the database due to network latency.

![][image2]

graph LR

    subgraph clients\[Clients\]  
        webapp\[Web App\]  
        mobile\[Mobile App\]  
        api\[API\]  
    end

    LB\[Load Balancer\]

    subgraph Backend Servers

       subgraph Server 1  
            S1\[Server 1\]  
            L1\[Local Cache\]  
         end

            subgraph Server 2  
                S2\[Server 2\]  
                L2\[Local Cache\]  
            end  
    end

    DB\[(Database)\]

    webapp \--\> LB  
    mobile \--\> LB  
    api \--\> LB

    LB \--\> S1  
    LB \--\> S2

    S1 \---\> DB  
    S2 \---\> DB

    S1 \-.-\> L1  
    S2 \-.-\> L2

Now the application servers are able to handle the load and the response time is well within the acceptable threshold. However, you are still seeing some performance issues.

* Cache misses \- You still notice that there are a lot of cache misses. This is because the requests are being distributed across the application servers. This means that the local cache on each server will only have a subset of the data.  
* Stale data \- The data in the local cache is not always up to date. Again this is because the requests are being distributed across the application servers and updates to the data are not being propagated to each local cache.  
* Memory consumption \- The local cache is consuming a lot of memory.  
  Your application servers are coupled with the local cache. This means that you cannot scale the application servers independently of the local cache or vice versa.

### **Global cache**

Due to the above issues, it becomes apparent that you need to move the local cache to a separate server. This will allow you to scale the application servers and the cache servers independently. This will also allow you to have a single source of truth for the data.

This is where a global cache comes in. The global cache is a separate server which stores the data. The application servers can query the database and the global cache to get the data. The global cache can also be updated by the application servers. This will allow the application servers to have a single source of truth for the data.

graph LR

    subgraph clients\[Clients\]  
        webapp\[Web App\]  
        mobile\[Mobile App\]  
        api\[API\]  
    end

    LB\[Load Balancer\]

    subgraph Backend Servers  
        S1\[Server 1\]  
        S2\[Server 2\]  
    end

    DB\[(Database)\]  
    GC\[Global Cache\]

    webapp \--\> LB  
    mobile \--\> LB  
    api \--\> LB

    LB \--\> S1  
    LB \--\> S2

    S1 \--\> DB  
    S2 \--\> DB

    S1 \-.-\> GC  
    S2 \-.-\> GC

### **Distributed cache**

There still exists some scope of further improvement:

* Single point of failure \- The global cache is a single point of failure. If the global cache goes down, the application servers might not be able to get the data.  
* Scalability \- The global cache is a single server. This means that it can only handle a certain amount of traffic. If the traffic increases, the global cache will not be able to handle it. Horizontal scaling wold require massive changes while vertical scaling might not be financially viable.

This is where a distributed cache comes in. The distributed cache is a cluster of servers which store the data.

However, the distributed cache is not a drop in replacement for the global cache. The distributed cache is a cluster of servers which store the data. Rather than implementing a distributed cache from scratch, you can use a third party solution like Redis, Memcached, etc.

## **Redis**

* Redis is an **in-memory** key-value data store that can be used as a database, cache, and message broker.  
  It shares the benefits of a local cache like **fast access** and **low latency** due to the data being stored in memory.  
* Optional durability \- Using volatile memory means that the data is lost when the server is restarted. However, Redis provides the option to persist the data to disk. This means that the data can be recovered even after a server restart.

### **Usage**

\!\!\! info “Redis is a key-value store”

Redis is a key-value store. This means that the data is stored as a key-value pair. The key is used to retrieve the value. The key can be any string. The value can be a string, a list, a set, a sorted set, a hash, or a bitmap.

\`\`\`redis  
SET key value  
GET key  
\`\`\`

\`\`\`redis  
SET user:1 "Tantia Tope"  
GET user:1  
\`\`\`

#### **Setup**

* Install \- brew install redis  
  * Find other installation options [here](https://hackmd.io/m53U71a2Qu6slyfYNIiIfQ).  
* Start server \- redis-server  
* Start client \- redis-cli

#### **Commands**

* SET \- SET key value

Set the value of a key.  
SET user:1 "Tantia Tope"

*   
* GET \- GET key

Get the value of a key.  
GET user:1  
\> "Tantia Tope"

*   
* EXISTS \- EXISTS key

Check if a key exists.  
EXISTS user:1  
\> 1  // 1 means true

*   
* DEL \- DEL key

Delete a key.  
DEL user:1  
\> 1  // 1 means true

*   
* INCR \- INCR key

Atomically increment the value of a key by 1\.  
INCR counter  
\> 1

*   
* CONFIG \- CONFIG SET parameter value

Set a configuration parameter to the given value.  
CONFIG SET maxmemory 1mb  
\> OK

* 

Try out the commands [here](https://try.redis.io/).

#### **Keys**

* Very long keys are not a good idea. For instance a key of 1024 bytes is a bad idea not only memory-wise, but also because the lookup of the key in the dataset may require several costly key-comparisons. Even when the task at hand is to match the existence of a large value, hashing it (for example with SHA1) is a better idea, especially from the perspective of memory and bandwidth.  
* Very short keys are often not a good idea. There is little point in writing “u1000flw” as a key if you can instead write “user:1000:followers”. The latter is more readable and the added space is minor compared to the space used by the key object itself and the value object. While short keys will obviously consume a bit less memory, your job is to find the right balance.  
* Try to stick with a schema. For instance “object-type:id” is a good idea, as in “user:1000”. Dots or dashes are often used for multi-word fields, as in “comment:4321:reply.to” or “comment:4321:reply-to”.  
* The maximum allowed key size is 512 MB.

## **Eviction policies**

Redis stores the data in memory, that is a limited resource. For applications running on high user base production environment, redis can run out of memory. This is where eviction policies come in. Eviction policies are used to determine which data to evict when the memory limit is reached.

By default, Redis uses the noeviction policy. This means that when the memory limit is reached, Redis will not evict any data. Instead, it will return an error.

127.0.0.1:6379\> CONFIG GET maxmemory-policy  
1) "maxmemory-policy"  
2) "noeviction"

How? As we have asked redis server to not restrict its memory use, and a noeviction policy means the data in redis would keep on growing, at some point, it reaches a critical level.

Redis can no longer add new records to it, as it gets limited by the memory available. The system kills redis as it gets choked by lack of memory and our apps begin to crash due to connection refused from redis.

Since redis is an in-memory solution, we have to increase the system’s RAM, and restart it to get it running again. Even though increasing RAM is a possible solution technically, this often is not a viable solution financially as RAM can cost a dime when we start scaling vertically. So then, there are disk based data store options which one can explore in these cases.

A cache store as the name suggests, is a solution for short-lived data which is safe to be purged in due course of time. Cache is used in applications that require high speed access to certain set of data on a regular basis. Cache is also used when an application want to store intermediate metadata of ongoing tasks for some hours or even days. This suggests that we can ask redis to remove unwanted or outdated records from its memory that would reduce the memory footprint of redis and reduce the chance of a downtime.

Redis provides a number of eviction policies to choose from. The eviction policy can be set using the maxmemory-policy configuration parameter.

### **LRU**

The LRU policy evicts the least recently used keys first.

127.0.0.1:6379\> CONFIG SET maxmemory-policy allkeys-lru  
OK

Let us also set the maxmemory to 1MB.

127.0.0.1:6379\> CONFIG SET maxmemory 1mb  
OK

To test the LRU policy, the redis-cli’s capabilities are limited. Hence, we will switch to a python script using the [redis-py](https://github.com/redis/redis-py) library.

Let us first create an instance of the redis client.

from redis import Redis

instance \= Redis(host=DEFAULT\_HOST, port=DEFAULT\_PORT)

Flush the database to remove any existing data.

instance.flushall()

Let’s test out creating a key-value pair.

instance.set('key', 'value')

Getting the value of the key.

instance.get('key')

\> b'value'

Let us check the memory usage of the redis instance.

print(f"Used memory: {node.info()\['used\_memory\_human'\]}")

\> Used memory: 1.49M

To simulate the LRU policy, we will be a little creative. We will create some key-value pairs first and then access a subset of them. Then, we will flood the redis instance with a lot of key-value pairs. This will cause the redis instance to reach the memory limit and start evicting keys.

According to the LRU policy, the least recently used keys will be evicted first. Hence, the keys that we accessed earlier will not be evicted.

Let us first create a utility function to create a key-value pair and access it.

def save\_data(end: int, start: int \= 0\) \-\> None:  
    for i in range(start, end):  
        instance.set(f"key-{i}",   
                      f"value-{i}")

def read\_data(end: int, start: int \= 0\) \-\> None:  
    for i in range(start, end):  
        instance.get(f"key-{i}")

Let us create the initial key-value pairs.

store\_data(redis, end=5000)

Read a subset of the keys.

read\_data(redis, end=1000)

Now, let us flood the redis instance with a lot of key-value pairs.

store\_data(redis, end=10000, start=5000)

Let us see if the keys that we accessed earlier are still present.

instance.get('key-1')

\> b'value-1'

instance.get('key-1000')

\> b'value-1000'

You could also use the mget command to get multiple keys at once.

instance.mget(\[f"key-{i}" for i in range(1, 1001)\])

Plotting the distribution of the keys that were evicted affirms the LRU policy.

![LRU][image3]

It can be noticed here that:

* There are lesser evictions for the keys that were accessed earlier.  
* The remaining keys in the initial suffered a lot of evictions.  
* The keys in the second set are evicted in the order they were created and hence most recent keys are retained.

Find out more about other eviction policies [here](https://redis.io/docs/reference/eviction/).

## **Cluster mode**

### **Motivation**

Currently, we have been working with a single redis instance. This is fine for development and testing purposes. However, for production, we need a more robust solution. Redis provides a cluster mode to achieve this.

As mentioned in the above sections, a distributed cache is preferred over a single instance cache. This is because a single instance cache is a single point of failure. If the cache goes down, the entire application will be affected. A distributed cache is a collection of cache instances. If one cache instance goes down, the other instances can still serve the requests.

Also, a single instance cache is limited by the memory available on the machine. A distributed cache can be scaled horizontally by adding more cache instances. This will increase the memory available to the cache.

### **Specification**

Each cluster node is a single Redis instance. The cluster nodes are connected to each other using a gossip protocol. This allows the cluster to automatically detect the failure of a node and reconfigure itself to ensure that the data is still available. Hence, ever cluster node requires two open TCP ports: a Redis TCP port used to serve clients, e.g., 6379, and second port known as the cluster bus port used to communicate with other nodes, e.g., 16379\.

#### **Sharding**

In cluster mode, Redis automatically partitions the data across multiple nodes. A way to shard the data uniformly is to use the consistent hashing algorithm. This algorithm maps a key to a node in the cluster. The hash of the key is used to determine the node. This ensures that the same key is always mapped to the same node. This is important as it ensures that the data is always available on the same node. This also ensures that the data is evenly distributed across the nodes.

However, this algorithm has a problem. If a node is added or removed from the cluster, the data will be redistributed across the nodes. This will cause a lot of data movement and will affect the performance of the cluster. Hence, **Redis does not use the consistent hashing algorithm.**

To solve the problem, Redis uses a concept called slot. A slot is a range of keys. Each node is responsible for a subset of the slots. The number of slots is fixed at 16384\. This means that each node is responsible for 16384/number of nodes keys. This ensures that the data is evenly distributed across the nodes. If a node is added or removed from the cluster, the data is redistributed across the nodes. However, this redistribution is limited to the slots that the node is responsible for. This ensures that the data movement is limited and the performance of the cluster is not affected.

Learn more about the sharding algorithm [here](https://severalnines.com/blog/hash-slot-vs-consistent-hashing-redis/).

#### **Replication**

Since the data is partitioned across multiple nodes, it is important to ensure that the data is replicated across the nodes. This ensures that the data is available even if a node goes down. Redis uses a master-slave replication model. The master node is responsible for the data. The slave nodes are responsible for replicating the data from the master node. If the master node goes down, one of the slave nodes is promoted to be the new master node. This ensures that the data is still available.

#### **Configuration**

The cluster mode is enabled by setting the cluster-enabled configuration option to yes. The default value is no.

To enable cluster mode, set the cluster-enabled directive to yes. Every instance also contains the path of a file where the configuration for this node is stored, which by default is nodes.conf. This file is never touched by humans; it is simply generated at startup by the Redis Cluster instances, and updated every time it is needed.

Note that the minimal cluster that works as expected must contain at least three master nodes.

This what our redis.conf file looks like.

port 6379  
cluster-enabled yes  
cluster-config-file nodes.conf  
cluster-node-timeout 5000  
appendonly yes

### **Setup**

To setup a cluster, we will use docker. We will create a docker network to connect the redis instances. This will allow the redis instances to communicate with each other.

\> docker network create redis-cluster

Check if the network was created.

\> docker network ls  
NETWORK ID     NAME              DRIVER    SCOPE  
a80462bf2a69   redis\_cluster     bridge    local

Now, we will create the redis instances. We will create 6 redis instances. 3 of them will be master nodes and the other 3 will be slave nodes.

You can create the instances using the following command.

\> docker run \-d \-v \\  
  redis.conf:/usr/local/etc/redis/redis.conf \\  
  \--name redis-\<node-number\> \--net redis-cluster \\  
  redis redis-server /usr/local/etc/redis/redis.conf

The above command will create a redis instance with the name redis-\<node-number\>. The configuration file is mounted to the container. The container is connected to the redis-cluster network.

You can either create the instances manually or write a small bash script to create the instances.

\#\!/usr/bin/env bash  
for ind in $(seq 1 6); do  
    docker run \-d \\  
        \-v redis.conf:/usr/local/etc/redis/redis.conf \\  
        \--name redis-$ind \\  
        \--net redis\_cluster \\  
        redis redis-server /usr/local/etc/redis/redis.conf  
done

This will create 6 redis instances and connect them to the redis-cluster network.

Check if the instances were created.

\> docker ps

You can use the inspect command to get the IP address of the instances.

\> docker inspect \-f '{{ (index .NetworkSettings.Networks "redis\_cluster").IPAddress }}' redis-1  
172.19.0.2

Currently, Redis Cluster does not support NATted environments and in general environments where IP addresses or TCP ports are remapped. Along with this, to run a Redis cluster using containers, we need a script distributed by Redis

docker run \-i \--rm \--net redis\_cluster ruby sh \-c '\\  
 gem install redis \\  
 && wget http://download.redis.io/redis-stable/src/redis-trib.rb \\  
 && ruby redis-trib.rb create \--replicas 1 \\  
 \<ip-address-1\>:6379 \<ip-address-2\>:6379 \\  
 ...

or you can use the redis-cli command as well

\> redis-cli \--cluster create  \<ip-address-1\>:6379 \\  
 ...  
\--cluster-replicas 1

Using the replicas option, we can specify the number of replicas for each master node. In our case, we have specified 1 replica for each master node and hence, we have 3 master nodes and 3 slave nodes.

It is just better to combine these steps in a bash script.

\#\!/usr/bin/env bash  
for ind in $(seq 1 6); do  
    docker run \-d \\  
        \-v redis.conf:/usr/local/etc/redis/redis.conf \\  
        \--name redis-$ind \\  
        \--net redis\_cluster \\  
        redis redis-server /usr/local/etc/redis/redis.conf  
done

echo 'yes' | docker run \-i \--rm \--net redis\_cluster ruby sh \-c '\\  
 gem install redis \\  
 && curl \-O http://download.redis.io/redis-stable/src/redis-trib.rb \\  
 && ruby redis-trib.rb create \--replicas 1 \\  
 '"$(for ind in $(seq 1 3); do  
    echo \-n "$(docker inspect \-f \\  
        '{{(index .NetworkSettings.Networks "redis\_cluster").IPAddress}}' \\  
        "redis-$ind")"':6379 '  
done)"

Your cluster is now ready. You can check the status of the cluster using the cluster info command.

\> docker exec \-it redis-1 redis-cli \-c cluster info

### **Testing the cluster**

To connect to the cluster, use the following command to connect to the container.

\> docker exec \-it redis-1 bash  
root@f7d86a9667e8:/data\# redis-cli  
127.0.0.1:6379\>

Set a key-value pair.

redis 127.0.0.1:6379\> set foo bar  
\-\> Redirected to slot \[12182\] located at 172.19.0.2:6379  
OK

The Redis Cluster client will automatically redirect the request to the correct node. You can check the node to which the request was redirected using the cluster nodes command.

\> cluster nodes

The above command will show the node to which the request was redirected.

You can also use the cluster slots command to get the slot to node mapping.

\> cluster slots

Adding another key-value pair could result in a different node being redirected to.

redis 127.0.0.1:7002\> set hello world  
\-\> Redirected to slot \[866\] located at 172.19.0.4:6379  
OK

Fetching the value of the key will also result in a redirection to the correct node where the key is stored.

redis 127.0.0.1:7000\> get foo  
\-\> Redirected to slot \[12182\] located at 172.19.0.2:6379  
"bar"  
redis 127.0.0.1:7002\> get hello  
\-\> Redirected to slot \[866\] located at 172.19.0.4:6379  
"world"

## **Code Glossary**

| Description | Link |
| :---- | :---- |
| redis.conf | [1](https://github.com/kanmaytacker/system-design/blob/master/guides/redis/redis.conf) |
| create\_cluster.sh | [2](https://github.com/kanmaytacker/system-design/blob/master/guides/redis/create_cluster.sh) |
| Python client | [3](https://github.com/kanmaytacker/system-design/blob/master/guides/redis/python/redis-cluster.py#L178) |

## **References**

* [Eviction Policies](https://redis.io/docs/reference/eviction/)  
* [How Redis eviction policies work](https://crypt.codemancers.com/posts/2021-05-21-understanding-how-redis-eviction-policies-work/)  
* [Redis Cluster](https://redis.io/topics/cluster-tutorial)  
* [Redis Cluster Specification](https://redis.io/topics/cluster-spec)  
* [Creating a Redis Cluster using Docker](https://medium.com/commencis/creating-redis-cluster-using-docker-67f65545796d)  
* [Building Redis cluster with Docker compose](https://pierreabreu.medium.com/building-redis-cluster-with-docker-compose-9569ddb6414a)

Select a repo  


[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAfQAAAH0CAYAAADL1t+KAACAAElEQVR4Xuy9h7cdxbH2ff+T7773fa+JQiIbZJOTiTYYg0GAwTZggi/RiGhARJEz2ICMRTI5CZAEQiQhEEkkIQmECCJnA5IQor9TfehN7WdXh+mZuTtM/dbaq09VVz/d0xPqzOzZM/9hlMYwZswY+/Exfvx485Of/MR89NFH1t54443NUUcdZf++4oorzOqrr26WLl3aip8+fbq56KKLzPfff29t1J85c6bVu+SSS1o+gjRJ20F/r7/++ubrr79u+aZOnWrbTps2zdqoTTh9KiWb4O1ovFR/zz33tOoJGh/5582bZ22KHzlypPn3v//dirn99tttzIsvvtjycebPn2/raZ44d9xxh/VT38Stt95q7bfffrstjnPuueeaESNGmHfeeaflo3nfdNNNzcEHH2ztt956y6ywwgp2XI7zzz/frLXWWq11xJd97ty5tt9//OMfrXjioIMOMscdd5z9G9fLY489ZttMmjSp5SOOOOIIM3r0aLNs2TLzxRdfmFVWWcWcddZZbTG///3vo8vpti9ax6RBy8057LDD7HbB18Nnn31m1lxzTXPGGWdYe8GCBbafW265pRVDy7PFFlvYv5977jlb/8wzz7Tq33zzTbutv/feey0fp8h2sueee7bqSZfPV8p6pPmhNsccc0wrxgfN12abbdbme/TRR83uu+/eNkfEtttua/bZZ5+WTX389re/ZRHGjB071owaNcr+TfveaqutZsaNG9cWc/zxx9u2bp9KWSfUZssttzTLly9vxdx5553mn//8Z8tW6uE/0KEMLlJS5Jx55pl25/3444+tzRP6s88+a1ZccUWz+eabm7PPPtvMmjXLfPfdd7x5h75LsC4pOzBx0N9HH300izD2YLDyyivbMRGoTWACR5vg7eggTpo4bndQvfbaa61N8TvttFNbzIwZM2zMU0891eZ3XHfddbaeH7wJSnorrbRSazlSEvrOO+9sfvOb39i2/HPSSSfZA6eDDtB//OMfWzatG5ecCb7s119/ve333XffbdUjuF4oSdM/DV999VXbOO69916r9corr5iHH37Y/v30008zJWMTbGw5qS9KPJT06G+eAAhKHIccckjHPOy3335m++23b8XRPOyxxx7278WLF9s5uvzyy61NY//pT39q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FodqQa+qWsSKFXTc+hYF/Yknnmhv5IE71OFmHLgpxkc/+tH2rnoQbgoExsknn9zekAg31Pjyl78879aceHKAV/y4YQbu2LbLLrvM7luNG3fgNsO44QtumIE7d8nds6qqqriqBb2qahHLKuh/+ctfmi222KLZbrvtmlWrVrXFGLcGRvuNN97YbLnllrNbZqKg43aauC/1Lbfc0t5VDIX7lFNOaftxz2n049aceBKw5557tu/TS0E/8cQTm4033rjloB/32sbyc889N8unqqqKo1rQq6oWsVDQ8aobr77lFfh6663XvsrGbTFxH+yjjz56dj9sCMUZRR5CIcYredxeVYRbau61117t33hFjlubilauXNneelMK+oYbbtjej1uE22/ifty6raqqiqNa0KuqFrFQ0I866qj2PtO47zU+3b7ZZps1991332wdXHbHJXXcP3qrrbZq75ONcZAUdH2JHfdx32233dq/cc9qvArXQsFHQQcXY/GKXp5QINZYY43m2GOPnTemqqqqv2pBr6paxAovuaMwo6Djkjsue6PobrLJJu375nilfvnllzcnnXTSagVdCwUdAW2++ebNCSecMK9/p512agv6ihUr2rEXXXRR+4RCx6OPPjpvTFVVVX/9f16T7VYRXA12AAAAAElFTkSuQmCC>