# Redis集群模式

前面已经介绍了Redis的主从模式、哨兵模式，现在开始介绍Redis的集群模式。不管是之前的主从模式还是哨兵模式，集群中的`master`节点都只有一个，它们能一定程度上提高集群的稳定性和读的性能，但是今天介绍的集群模式它能让集群的稳定性进一步提高，同时还能提升集群的读能力。

## 复制和高可用

Redis集群模式它提供了主从复制功能，其中每个Redis服务被叫做一个node，其中node又区分`master node`和`replica/slave node`。其中`master node`主要负责处理客户端的写请求，而`slave node`负责处理客户端的读请求。

Redis集群除了复制功能之外还提供了类似于哨兵模式的功能，用来提高集群的高可用性。Redis集群中各个节点将互相监视各自的运行状态，并且主节点下线是，将提升该节点的从节点作为新的主节点继续提供服务。

## 分片和重分片

在主从模式和哨兵模式中所有的Redis服务器存储的数据是一致的，写数据时将数据写入的`master`节点，然后`slave`节点从`master`节点复制数据，读数据的时候则从`slave`节点读取。这种做法有以下缺点：

- 所有的写都在一台服务器上，而这台服务器的写的性能决定了整个集群写的性能。

- 每台服务器上存储的数据都是一样的，浪费了服务器的内存资源。

而在集群模式中则不是如此，集群模式通过将数据库分散存储到多个节点上来平衡各个节点的负载压力。简单的说，Redis集群会将整个数据库空间划分成`16384`个槽来实现数据分片，而集群中的各个主节点则会负责处理所有槽中的一部分。当用户尝试将一个键值存储到Redis集群中时，客户端会先计算出键所属的槽，然后在记录集群节点分槽的映射表中找到对应槽的节点，最后将键值数据存储到对应的节点中。其大致结构如下图所示：

<img title="" src="http://img.bakelu.cn/blog/redis%E9%9B%86%E7%BE%A4%E5%88%86%E7%89%87.png" alt="" data-align="center">

当集群需要扩张时，只需要向redis发送几条指令即可。集群会将相应的槽和槽中存储的数据迁移至新的节点中即可。当集群需要缩减时，其与扩张的思想类似。被移除的节点也会将自己负责处理的槽已经槽中的数据转交给集群中的其他节点复制。最重要的一点是，无论是扩张还是缩减，整个重分片的过程都可以在线进行，Redis集群无需因此而停机维护。

## 集群搭建

> 环境说明：本实验是在redis`5.0.12`环境中进行的，对于不同的版本可能会存在稍许不同。

以下示例是在同一台服务器上进行的，在实际生产环境中最好一个Redis服务一台服务器。本集群由3个主节点和3个从节点构成，其端口从`6379~6384`。

### 启动Redis服务

根据前面的架构信息，我们这里一共需要6个redis服务，其中3主3从。在实际生产环境中我们也应该使用单数个主节点，并且每个主节点至少有1个以上的从节点。

#### 修改配置文件

首先我们需要修改Redis配置文件，其配置文件如下：

```
# 继承使用的源码中的配置
include redis.conf

# ip和端口已经开启后台运行模式
bind 192.168.1.7
port 6379
daemonize yes

# pid、日志、rdb、aof文件配置
pidfile "/var/run/redis-6379.pid"
logfile "redis-6379.log"
dbfilename "dump-6379.rdb"
dir "."
appendfilename "appendonly-6379.aof"

# redis服务密码和集群密码
requirepass "redis"
masterauth "redis"

# 开启集群模式
cluster-enabled yes
# 集群配置文件
cluster-config-file nodes-6379.conf
```

我这里继承了源码中的redis配置文件，该文件中的配置会覆盖`redis.conf`中的配置。我们一个创建6个配置文件，其名称为`redis-${port}.conf`，并修改其中`${port}`相关配置。

#### 启动

配置文件配置好之后我们分别启动每个Redis服务。

```shell
redis-server redis-${port}.conf
```

其中的`${port}`代表的不同端口的redis服务。全部启动之后我们可以通过以下命令查看6个节点是否都已经启动成功。

```shell
ps -ef | grep redis
```

该命令会打印出进程同时筛选出进程名字中含有`redis`关键字的记录。

### 加入集群

所有Redis节点启动没问题之后我们可以使用`redis-cli`工具连接上Redis服务。

```shell
redis-cli -h ${host} -p ${port} -a ${password}
```

我这里连接的是6379服务器，连接上Redis之后可以通过下面命令查看集群节点：

```shell
cluster nodes
```

其打印结果如下：

```log
9f65603fd6233332f69099097ed36725b68fec66 192.168.1.7:6379@16379 myself,master - 0 1668583719000 5 connected
```

打印结果显示目前集群只有一个节点，这时我们需要将另外5个节点加入到一个集群中。这里我们将另外5个节点加入到`6379`节点所在集群。在`redis-cli`中执行以下命令：

```shell
cluster meet ${host} ${port}
```

执行完上面的命令后，所有节点都在一个集群中了。我们可以再次通过`cluster nodes`查看集群节点信息。

```log
192.168.1.7:6379> cluster nodes
9f5676920b848a1b9ce19acaa83e1261a20a712a 192.168.1.7:6381@16381 master - 0 1668583865000 2 connected
9f65603fd6233332f69099097ed36725b68fec66 192.168.1.7:6379@16379 myself,master - 0 1668583865000 5 connected
73b87533643becc8624b9d3456786f06188825b4 192.168.1.7:6383@16383 master - 0 1668583864000 4 connected
8047bdf6bb94f09e1bba8d2d6cf1fcf6330ca4f6 192.168.1.7:6380@16380 master - 0 1668583864064 1 connected
e231c30fa07e741a26eb1010a42b06230ca5b138 192.168.1.7:6384@16384 master - 0 1668583866099 0 connected
1906807f5fb29a0ec1b047c0b52583c081ad690e 192.168.1.7:6382@16382 master - 0 1668583865080 3 connected
```

上面一共打印出6个节点的信息，并且每个节点都是我们配置的节点，说明所有节点都已经在一个集群中了。

### 设置slave节点

上面所有的节点都已经在同一个集群了，但是所有节点都是`master`节点。现在我们需要将部分节点设置成`slave`节点。这里我将按照如下划分逻辑：

| 主节点              | 从节点              |
|:----------------:|:----------------:|
| 192.168.1.7:6379 | 192.168.1.7:6380 |
| 192.168.1.7:6381 | 192.168.1.7:6382 |
| 192.168.1.7:6383 | 192.168.1.7:6384 |

我们可以通过如下命令设置从节点：

```shell
redis-cli -h ${host} -p ${port} -a ${password} cluster replicate ${node-id}
```

> 注意：这里面是在Linux服务器的命令行执行，不是在redis-cli的命令行执行。我这里是为了不想一个个登录到对应的从节点上去。如果你登录到对应的从节点上，可以直接使用命令：
> 
> ```shell
> cluster replicate ${node-id}
> ```

这里面的`${host}`和`${port}`是从节点的IP和端口，其中`-a`后面是密码，`${node-id}`是主节点的nodeID。对应到我们刚才的信息中，命令如下：

```shell
redis-cli -h 192.168.1.7 -p 6380 -a redis cluster replicate 9f65603fd6233332f69099097ed36725b68fec66
redis-cli -h 192.168.1.7 -p 6382 -a redis cluster replicate 9f5676920b848a1b9ce19acaa83e1261a20a712a
redis-cli -h 192.168.1.7 -p 6384 -a redis cluster replicate 73b87533643becc8624b9d3456786f06188825b4
```

通过上面的操作，我们再次查看节点信息，其打印结果如下：

```log
192.168.1.7:6379> cluster nodes
9f5676920b848a1b9ce19acaa83e1261a20a712a 192.168.1.7:6381@16381 master - 0 1668584149000 2 connected 5001 10000
9f65603fd6233332f69099097ed36725b68fec66 192.168.1.7:6379@16379 myself,master - 0 1668584148000 5 connected 0 5000
73b87533643becc8624b9d3456786f06188825b4 192.168.1.7:6383@16383 master - 0 1668584147481 4 connected 10001 16383
8047bdf6bb94f09e1bba8d2d6cf1fcf6330ca4f6 192.168.1.7:6380@16380 slave 9f65603fd6233332f69099097ed36725b68fec66 0 1668584148000 5 connected
e231c30fa07e741a26eb1010a42b06230ca5b138 192.168.1.7:6384@16384 slave 73b87533643becc8624b9d3456786f06188825b4 0 1668584148498 4 connected
1906807f5fb29a0ec1b047c0b52583c081ad690e 192.168.1.7:6382@16382 slave 9f5676920b848a1b9ce19acaa83e1261a20a712a 0 1668584149518 3 connected
```

### 分配槽位

通过上面操作我们已经将所有节点加入到一个集群中了，同时还设置好了主从节点，但是此时集群还不能正常使用，我们还缺少了重要的一步分配槽位。前面我们讲过，我们需要将`16384`个槽位分配到不同的节点。

这里我们可以通过`cluster addslots`设置槽位，我们的划分规则如下：

| 节点               | 开始槽位  | 结束槽位  |
| ---------------- | ----- | ----- |
| 192.168.1.7:6379 | 0     | 5000  |
| 192.168.1.7:6381 | 5001  | 10000 |
| 192.168.1.7:6383 | 10001 | 16383 |

命令如下：

```shell
redis-cli -h 192.168.1.7 -p 6379 -a redis cluster addslots {0..5000}
redis-cli -h 192.168.1.7 -p 6381 -a redis cluster addslots {5001..10000}
redis-cli -h 192.168.1.7 -p 6383 -a redis cluster addslots {10001..16383}
```

`cluster addslots`命令格式是不支持范围的，例如你现在要指定`1,2,3,4,5`这5个槽位，你在`redis-cli`中需要按如下方式写：

```shell
cluster addslots 1 2 3 4 5
```

我们这里是在linux的bash中写，它可以简写成如下：

```shell
cluster addslots {1..5}
```

通过上面的方式我们就将所有的槽位信息划分到3个master节点中了，我们再次查看节点信息，其结果如下：

```log
192.168.1.7:6379> cluster nodes
9f5676920b848a1b9ce19acaa83e1261a20a712a 192.168.1.7:6381@16381 master - 0 1668584149000 2 connected 5001 10000
9f65603fd6233332f69099097ed36725b68fec66 192.168.1.7:6379@16379 myself,master - 0 1668584148000 5 connected 0 5000
73b87533643becc8624b9d3456786f06188825b4 192.168.1.7:6383@16383 master - 0 1668584147481 4 connected 10001 16383
8047bdf6bb94f09e1bba8d2d6cf1fcf6330ca4f6 192.168.1.7:6380@16380 slave 9f65603fd6233332f69099097ed36725b68fec66 0 1668584148000 5 connected
e231c30fa07e741a26eb1010a42b06230ca5b138 192.168.1.7:6384@16384 slave 73b87533643becc8624b9d3456786f06188825b4 0 1668584148498 4 connected
1906807f5fb29a0ec1b047c0b52583c081ad690e 192.168.1.7:6382@16382 slave 9f5676920b848a1b9ce19acaa83e1261a20a712a 0 1668584149518 3 connected
```

从上面的打印结果可以看出，`master`节点后面以及附带了槽位信息。同时我们还可以使用`cluster info`来查看集群信息。

```log
192.168.1.7:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:5
cluster_my_epoch:5
cluster_stats_messages_ping_sent:4504
cluster_stats_messages_pong_sent:4115
cluster_stats_messages_meet_sent:5
cluster_stats_messages_sent:8624
cluster_stats_messages_ping_received:4115
cluster_stats_messages_pong_received:3912
cluster_stats_messages_received:8027
```

从上面的打印消息可以看出，集群当前的状态是正常的。

### 测试

我们在`redis-cli`中尝试写入一个数据测试，看看集群是否正常。

```log
192.168.1.7:6379> set k1 v1
(error) MOVED 12706 192.168.1.7:6383
```

结果提示写入异常，提示你需要去`6383`的节点执行写入数据。这是因为我们之前使用`redis-cli`连接到redis服务的命令缺少了`-c`参数，接下来我们使用如下命令从新连接上redis服务。

```shell
redis-cli -h 192.168.1.7 -p 6379 -a redis -c
```

再次执行数据写入，其响应结果如下：

```shell
192.168.1.7:6379> set k1 v1
-> Redirected to slot [12706] located at 192.168.1.7:6383
OK
192.168.1.7:6383> get k1
"v1"
```

## 另一种简便方式搭建

上面的方式相对来说比较繁琐，启动所有节点之后需要我们将所有节点加入集群，然后设置主从节点最后分配槽位。实际上我们可以使用另外一种更加简便的方式，让redis自己为我们构建集群。

### 启动Redis节点

该步骤还是和之前的步骤一样，配置好配置文件之后，然后启动节点，这里就不再赘述了。

### 构建集群

接下来我们通过如下命令直接构建集群即可，其格式如下：

```shell
redis-cli --cluster create ${host}:port --cluster-replicas ${replica-number}
```

其中`${host}:port`是节点的ip和端口，可以写多个。`--cluster-replicas`用来指定主节点的从节点个数。对应到我们的环境中，其命令如下：

```shell
redis-cli -a redis --cluster create 192.168.1.7:6379 192.168.1.7:6380 192.168.1.7:6381 192.168.1.7:6382 192.168.1.7:6383 192.168.1.7:6384 --cluster-replicas 1
```

执行完上面的指令之后，命令行中会打印出集群配置方案，我们只需要输入`yes`同意方案即可。最后的打印结果如下：

```log
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.1.7:6383 to 192.168.1.7:6379
Adding replica 192.168.1.7:6384 to 192.168.1.7:6380
Adding replica 192.168.1.7:6382 to 192.168.1.7:6381
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 67461c7609453a969d76ad724eb3151fb65b4bf1 192.168.1.7:6379
   slots:[0-5460] (5461 slots) master
M: 338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 192.168.1.7:6380
   slots:[5461-10922] (5462 slots) master
M: 71b1d3a4c717c6b676bbc39773c447e996a31401 192.168.1.7:6381
   slots:[10923-16383] (5461 slots) master
S: ef4b87179f58866314099c9a66626c5987ab274c 192.168.1.7:6382
   replicates 67461c7609453a969d76ad724eb3151fb65b4bf1
S: abb27e4e8a8348a5db9f901dd662074131df0ca5 192.168.1.7:6383
   replicates 338b6fbd64e5bd0db1be06c8d68d7bf1355716a6
S: cbadff3e32eb7d82ae2682b3216418a597aae93d 192.168.1.7:6384
   replicates 71b1d3a4c717c6b676bbc39773c447e996a31401
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.....
>>> Performing Cluster Check (using node 192.168.1.7:6379)
M: 67461c7609453a969d76ad724eb3151fb65b4bf1 192.168.1.7:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: cbadff3e32eb7d82ae2682b3216418a597aae93d 192.168.1.7:6384
   slots: (0 slots) slave
   replicates 71b1d3a4c717c6b676bbc39773c447e996a31401
S: ef4b87179f58866314099c9a66626c5987ab274c 192.168.1.7:6382
   slots: (0 slots) slave
   replicates 67461c7609453a969d76ad724eb3151fb65b4bf1
S: abb27e4e8a8348a5db9f901dd662074131df0ca5 192.168.1.7:6383
   slots: (0 slots) slave
   replicates 338b6fbd64e5bd0db1be06c8d68d7bf1355716a6
M: 338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 192.168.1.7:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 71b1d3a4c717c6b676bbc39773c447e996a31401 192.168.1.7:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

相比与我们之前复杂的过程，该方式更加的快速和高效。

## 添加节点

通过简便方式我们已经构建好了集群，此时集群的状态如下所示：

<img title="" src="http://img.bakelu.cn/blog/%E5%8A%A0%E5%85%A5%E8%8A%82%E7%82%B9%E5%89%8D.png" alt="" data-align="center">

> 注意：因为我这边测试过主节点宕机重启，所以目前主从节点与之前构建好的集群主从分配存在稍许不同。

现在我们需要向集群添加一主一从两个节点，下面我们将介绍如何向集群添加节点。

### 创建新的节点

和之前创建新节点的方式一样，我们拷贝之前`redis-6379.conf`文件两份分别命名为`redis-6385.conf`和`redis-6386.conf`，然后修改其中的端口为当前文件名字的端口，最后启动两个新的节点。

### 加入集群

通过如下命令，将`6385`和`6386`加入到集群。

```shell
redis-cli --cluster add-node 192.168.1.7:6385 192.168.1.7:6379 -a redis
redis-cli --cluster add-node 192.168.1.7:6386 192.168.1.7:6379 -a redis
```

此时可以查看集群状态如下：

```log
192.168.1.7:6379> cluster nodes
cbadff3e32eb7d82ae2682b3216418a597aae93d 192.168.1.7:6384@16384 master - 0 1668593820000 7 connected 10923-16383
8f5869c095955a29a632d1104202737ecb6fa3c5 192.168.1.7:6386@16386 master - 0 1668593819000 0 connected
ef4b87179f58866314099c9a66626c5987ab274c 192.168.1.7:6382@16382 slave 67461c7609453a969d76ad724eb3151fb65b4bf1 0 1668593821000 4 connected
abb27e4e8a8348a5db9f901dd662074131df0ca5 192.168.1.7:6383@16383 slave 338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 0 1668593820000 5 connected
027d0c50d5927cf0341915639abfd3ea58fa2171 192.168.1.7:6385@16385 master - 0 1668593821886 8 connected
338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 192.168.1.7:6380@16380 master - 0 1668593819000 2 connected 5461-10922
71b1d3a4c717c6b676bbc39773c447e996a31401 192.168.1.7:6381@16381 slave cbadff3e32eb7d82ae2682b3216418a597aae93d 0 1668593820868 7 connected
67461c7609453a969d76ad724eb3151fb65b4bf1 192.168.1.7:6379@16379 myself,master - 0 1668593819000 1 connected 0-5460
```

从结果可以看出两个新节点都加入了集群并且还是`master`节点，但是它们分配的槽都是0。那么接下来我们将`6386`设置成`6385`的从节点。

```shell
redis-cli -h 192.168.1.7 -p 6386 -a redis cluster replicate 027d0c50d5927cf0341915639abfd3ea58fa2171
```

### 重新分配槽

新节点设置完主从之后，我们接下来就是将部分槽分配给新添加的节点，执行以下命令即可：

```shell
redis-cli --cluster reshard 192.168.1.7:6385 -a redis
```

执行完命令之后，将会出现以下选项？

```log
How many slots do you want to move (from 1 to 16384)?
```

该选项是问你分配多少个槽，我这里是输入的`4208`，实际上你想分配多少个取决于你自己。

```log
What is the receiving node ID?
```

这个是问你哪个节点接受上面的槽，这里我们直接填`6385`的nodeId即可。

```log
Please enter all the source node IDs.
Type ‘all’ to use all the nodes as source nodes for the hash slots.
Type ‘done’ once you entered all the source nodes IDs.
```

这里是问你槽的来源，你可以指定一个节点或者多个节点，也可以是从全部的节点里面来。我这里直接输入的是`all`，表示从所有的节点中来。

```log
How many slots do you want to move (from 1 to 16384)? 4208
What is the receiving node ID? 027d0c50d5927cf0341915639abfd3ea58fa2171
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all
```

等待命令执行完成之后，再次查看节点信息，结果如下：

```log
192.168.1.7:6379> cluster nodes
cbadff3e32eb7d82ae2682b3216418a597aae93d 192.168.1.7:6384@16384 master - 0 1668594138000 7 connected 12325-16383
8f5869c095955a29a632d1104202737ecb6fa3c5 192.168.1.7:6386@16386 slave 027d0c50d5927cf0341915639abfd3ea58fa2171 0 1668594139000 8 connected
ef4b87179f58866314099c9a66626c5987ab274c 192.168.1.7:6382@16382 slave 67461c7609453a969d76ad724eb3151fb65b4bf1 0 1668594138000 4 connected
abb27e4e8a8348a5db9f901dd662074131df0ca5 192.168.1.7:6383@16383 slave 338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 0 1668594136803 5 connected
027d0c50d5927cf0341915639abfd3ea58fa2171 192.168.1.7:6385@16385 master - 0 1668594139857 8 connected 0-1401 5461-6863 10923-12324
338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 192.168.1.7:6380@16380 master - 0 1668594139000 2 connected 6864-10922
71b1d3a4c717c6b676bbc39773c447e996a31401 192.168.1.7:6381@16381 slave cbadff3e32eb7d82ae2682b3216418a597aae93d 0 1668594140870 7 connected
67461c7609453a969d76ad724eb3151fb65b4bf1 192.168.1.7:6379@16379 myself,master - 0 1668594138000 1 connected 1402-5460
```

<img title="" src="http://img.bakelu.cn/blog/%E5%8A%A0%E5%85%A5%E8%8A%82%E7%82%B9%E5%90%8E.png" alt="" data-align="center">

从上面结果可以看出，新加的节点分配新的槽`0-1401 5461-6863 10923-12324`。

## 移除节点

上面我们已经成功的添加节点了，接下来就是移除节点。这里我们移除掉`6385`和`6386`两个节点，这两个节点一个是主节点一个是从节点，这里我们优先移除从节点。从节点移除成功之后再移除主节点。如果主节点先移除再移除从节点，则会出现故障转移。

### 移除从节点

使用如下命令移除从节点：

```shell
redis-cli --cluster del-node 192.168.1.7:6386 8f5869c095955a29a632d1104202737ecb6fa3c5 -a redis
```

通过`cluster nodes`查看可以发现该节点已经不存在了。

### 主节点槽转移

由于`6385`是主节点，我们不能直接将其移除。如果直接移除了，那么该节点槽中的数据也就没了，集群中将存在未分配的槽，集群将变得不可用。所以在移除该节点前需要先将该节点中槽转移到其他节点上。

槽转移过程和新增加节点类似，我这将`6385`上所有槽转移到节点`6379`上去。接下就是执行`reshard`操作。

```shell
redis-cli --cluster reshard 192.168.1.7:6385 -a redis
```

执行完该操作后，会出现如下选择：

```log
How many slots do you want to move (from 1 to 16384)? 4208
What is the receiving node ID? 67461c7609453a969d76ad724eb3151fb65b4bf1
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 027d0c50d5927cf0341915639abfd3ea58fa2171
Source node #2: done
```

执行完上述操作之后，`6385`中已经没有数据了，移除该节点即可。

```shell
redis-cli --cluster del-node 192.168.1.7:6385 027d0c50d5927cf0341915639abfd3ea58fa2171 -a redis
```

再次查看集群状态结果如下：

```log
192.168.1.7:6379> cluster nodes
cbadff3e32eb7d82ae2682b3216418a597aae93d 192.168.1.7:6384@16384 master - 0 1668597857000 7 connected 12325-16383
ef4b87179f58866314099c9a66626c5987ab274c 192.168.1.7:6382@16382 slave 67461c7609453a969d76ad724eb3151fb65b4bf1 0 1668597857000 9 connected
abb27e4e8a8348a5db9f901dd662074131df0ca5 192.168.1.7:6383@16383 slave 338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 0 1668597857895 5 connected
338b6fbd64e5bd0db1be06c8d68d7bf1355716a6 192.168.1.7:6380@16380 master - 0 1668597858908 2 connected 6864-10922
71b1d3a4c717c6b676bbc39773c447e996a31401 192.168.1.7:6381@16381 slave cbadff3e32eb7d82ae2682b3216418a597aae93d 0 1668597857000 7 connected
67461c7609453a969d76ad724eb3151fb65b4bf1 192.168.1.7:6379@16379 myself,master - 0 1668597855000 9 connected 0-6863 10923-12324
```

<img title="" src="http://img.bakelu.cn/blog/%E7%A7%BB%E9%99%A4%E8%8A%82%E7%82%B9%E5%90%8E.png" alt="" data-align="center">

# 
