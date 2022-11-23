# Redis哨兵模式

## 简述

在之前的文章中介绍了Redis的主从复制，通常在使用主从模式时我们可以通过主节点写，从节点读的方式来实现读写分离提高系统性能，同时主从复制能一定程度上的防止缓存数据丢失。但是还是存在一个问题，如果主节点因为不明原因宕机导致不可用，那么在Redis主节点没有正常启动前，Redis将无法提供数据写入。

对于Redis主节点宕机的情况，通常情况下我们有两种选择。其一是人工处理完故障后使Redis主节点恢复正常。另外一种方式则是选择一个新的从节点作为主节点，其他从节点修改从新的主节点同步数据。需要注意的是，选择新的从节点数据可能与之前的主节点的数据不一致，如果对数据有强一致性要求则只能等主节点重启，不能使用第二种方案。

## sentinel

Redis提供的sentinel也就是我们常说的`哨兵模式`，它的实现实际上跟我们说的第二种方式一样。哨兵模式整体架构如下：

<img title="" src="http://img.bakelu.cn/blog/%E5%93%A8%E5%85%B5%E6%A8%A1%E5%BC%8F.png" alt="" data-align="center">

在上面的架构中，如果`redis master`发生故障而宕机了，那么哨兵会从`slave1`或者`slave2`中选出一个新的节点做为`master`节点，然后另外一个`slave`节点将从新选出的`master`节点上同步数据。

### 配置哨兵集群

在前面的图中我们可以看到，sentinel可以是一个集群。在实际生产应用中，几乎不可能使用单节点哨兵模式。其理由如下：

- 首先单节点哨兵如果宕机那么哨兵模式就没用了。在此情况下如果`master`故障，那么哨兵将不起作用。

- 如果哨兵跟`master`节点网络不稳定，很容易出现`master`节点无故障但是哨兵却做故障转移。

下面将介绍如何搭建一个sentinel集群。

#### sentinel配置文件

在redis的源码包中有一份`sentinel.conf`文件，在实际使用时我们可以直接拷贝该文件然后根据我们实际情况做部分修改即可。下面是对必要配置的介绍

```
# sentinel服务监听的端口
port 26379

# 指定IP，可以指定多个
bind 192.168.1.7

# sentinel密码
requirepass sentinel 

# 是否后台运行
daemonize yes

# pid文件
pidfile /var/run/redis-sentinel-26379.pid

# 日志文件
logfile "sentinel-26379.log"

# 目录
dir /tmp

# 监视的Redis主节点
sentinel monitor <master-name> <ip> <redis-port> <quorum>

# Redis集群密码
sentinel auth-pass <master-name> <password>

# 主节点或副本在指定时间内没有回复PING，便认为该节点为主观下线 S_DOWN 状态
sentinel down-after-milliseconds <master-name> <milliseconds>


# 在故障转移期间，多少个副本节点进行数据同步
# sentinel parallel-syncs <master-name> <numreplicas>

# 指定故障转移超时（以毫秒为单位）
# sentinel failover-timeout <master-name> <milliseconds>
```

#### 哨兵集群

根据上面的配置文件，我们现在构建一个拥有3个节点的哨兵集群。因为我这个是演示就直接在一台服务器上搭建了，其端口分别为`26379`、`26380`、`26381`。

`sentinel-26379.conf`文件内容如下：

```
port 26379
bind 192.168.1.7
requirepass sentinel
daemonize yes
pidfile /var/run/redis-sentinel-26379.pid
logfile "sentinel-26379.log"
dir /tmp

sentinel monitor mymaster 192.168.1.7 6379 2
sentinel auth-pass mymaster redis
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

后续`sentinel-26380.conf`和`sentinel-26381.conf`配置文件基本一样，只需要将`26379`修改成对应的端口即可。

`sentinel monitor <master-name> <ip> <redis-port> <quorum>`其中`<master-name>`代表的是redis集群的名字，`<ip>`则是主节点的ip地址，`<redis-port>`代表的是主节点的IP地址。`<quorum>`法定人数，该参数用于指定判断这个主服务器下线所需要的哨兵数量。

`sentinel auth-pass <master-name> <password>`该配置时用来指定集群的Redis访问密码的。

#### 启动redis主从复制

此处我们实验的Redis集群由3个节点组成，它们的端口分别为`6379`、`6380`、`6381`。需要注意的是本示例中主从节点都设置了密码，即如下配置：

```
requirepass "redis"
```

对于从节点我们应该配置访问主节点的密码，配置如下：

```
masterauth "redis"
```

主节点未进行该配置可以正常启动没问题，但是这里建议给主节点也添加上该配置。因为如果主节点故障转移之后作为从节点再次加入集群，如果没有配置该密码将无法进行数据复制。

最后就是从节点增加主节点配置即可。

```
replicaof 192.168.1.7 6379
```

依次启动Redis节点之后，我们就得到了一主(6379)两从(6380和6381)的主从集群了。我们可以通过`redis-cli`登录到master节点查看主从复制状态。

```shell
$ redis-cli -h 192.168.1.7
192.168.1.7:6379> auth redis
OK
192.168.1.7:6379> role
1) "master"
2) (integer) 56
3) 1) 1) "192.168.1.7"
      2) "6380"
      3) "56"
   2) 1) "192.168.1.7"
      2) "6381"
      3) "56"
```

#### 启动sentinel节点

启动sentinel节点很简单，可以通过`redis-sentinel`启动：

```shell
$ redis-sentinel sentinel-26379.conf
```

同样也可以使用`redis-server`增加`--sentinel`参数来启动。

```shell
$ redis-server sentinel-26380.conf --sentinel
```

实际上sentinel也就是一个redis服务。我们可以通过`redis-cli`登录到哨兵节点查看状态。

```shell
[root@localhost ~]$ redis-cli -h 192.168.1.7 -p 26379
192.168.1.7:26379> auth sentinel
OK
192.168.1.7:26379> sentinel masters
1)  1) "name"
    2) "mymaster"
    3) "ip"
    4) "192.168.1.7"
    5) "port"
    6) "6379"
    7) "runid"
    8) "39ca04e3f90edd77737dc62d1437941935974579"
    9) "flags"
   10) "master"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "447"
   19) "last-ping-reply"
   20) "447"
   21) "down-after-milliseconds"
   22) "30000"
   23) "info-refresh"
   24) "253"
   25) "role-reported"
   26) "master"
   27) "role-reported-time"
   28) "120611"
   29) "config-epoch"
   30) "0"
   31) "num-slaves"
   32) "2"
   33) "num-other-sentinels"
   34) "2"
   35) "quorum"
   36) "2"
   37) "failover-timeout"
   38) "180000"
   39) "parallel-syncs"
   40) "1"
```

通过`sentinel masters`命令我们可以查看所有监控的redis集群状态，从上面的结果可以看出，当前的master节点是`192.168.1.7:6379`，然后它拥有两个`slave`节点，同样它哨兵集群还包含了两个哨兵节点。

#### sentinel管理命令

sentinel提供了一些常用的操作指令用来管理集群。

- `sentinel masters`该指令用来打印出所有监控的redis集群信息。需要注意的是，一个哨兵集群可以同时监控多个redis集群。

- `sentinel master <master-name>`该指令用来获取指定redis集群信息。

- `sentinel slaves <master-name>`该指令用来返回集群的从节点。

- `sentinel sentinels <master-name>`该指令用来获取其他sentinel的相关信息。

- `sentinel get-master-addr-by-name <master-name>`获取给定主服务器的IP地址及端口。

- `sentinel reset <master-name-pattern>`该命令用来重置给定匹配的主服务器。该操作会使sentinel清理被匹配的服务器的相关信息同时还会遗忘掉匹配的主服务器已有的从服务器已经监视主服务器的sentinel，之后将从新搜索slave节点和sentinel。

- `sentinel failover <master-name>`强制目标服务器进行故障转移。

- `sentinel ckquorum <master-name>`检查可用sentinel的数量。该命令可以用来检查sentinel网络当前可用的sentinel数量是否达到了判断主服务器客观下线并实施故障转移所需的数量。

- `sentinel flushconfig`强制写入配置文件。sentinel在被监视的服务器发生变化时就会自动重写配置文件，这个命令可以在配置文件因为某些原因丢失或者错误时立即生成一个新的配置文件。同时当sentinel的配置项发生改变时，其内部也会用这个命令来创建新的配置文件替换旧的配置文件。

## 测试故障转移

在上面我们已经搭建好了拥有三个哨兵的一主两从的Redis集群，现在我们测试一下哨兵模式是否能够正常的进行故障转移。

### 停止当前master节点

这里我们首先停掉当前的master节点，也就是`192.168.1.7:6379`节点。可以看见sentinel日志中打印如下内容：

```log
57328:X 15 Nov 2022 16:29:27.725 # +sdown master mymaster 192.168.1.7 6379
57328:X 15 Nov 2022 16:29:27.818 # +new-epoch 1
57328:X 15 Nov 2022 16:29:27.819 # +vote-for-leader 608d7cefe3c72b8fbeb07bcd5dee2b5323742c88 1
57328:X 15 Nov 2022 16:29:28.764 # +config-update-from sentinel 608d7cefe3c72b8fbeb07bcd5dee2b5323742c88 192.168.1.7 26381 @ mymaster 192.168.1.7 6379
57328:X 15 Nov 2022 16:29:28.764 # +switch-master mymaster 192.168.1.7 6379 192.168.1.7 6380
57328:X 15 Nov 2022 16:29:28.764 * +slave slave 192.168.1.7:6381 192.168.1.7 6381 @ mymaster 192.168.1.7 6380
57328:X 15 Nov 2022 16:29:28.764 * +slave slave 192.168.1.7:6379 192.168.1.7 6379 @ mymaster 192.168.1.7 6380
57328:X 15 Nov 2022 16:29:58.828 # +sdown slave 192.168.1.7:6379 192.168.1.7 6379 @ mymaster 192.168.1.7 6380
```

从日志中可以看出sentinel判断出`6379`节点宕机了，然后选出了一个新的master节点`6380`当做新的主节点。

### 重启故障节点

我们再次重启`6379`从节点，`sentinel`打印日志如下：

```log
57328:X 15 Nov 2022 16:41:43.609 # -sdown slave 192.168.1.7:6379 192.168.1.7 6379 @ mymaster 192.168.1.7 6380
```

执行`sentinel slaves <master-name>`指令可以查看当前的从节点状态如下：

```log
192.168.1.7:26379> sentinel slaves mymaster
1)  1) "name"
    2) "192.168.1.7:6381"
    3) "ip"
    4) "192.168.1.7"
    5) "port"
    6) "6381"
    7) "runid"
    8) "c879cfcba138ff67fb1e11c5a08fca4adbec7fca"
    9) "flags"
   10) "slave"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "381"
   19) "last-ping-reply"
   20) "381"
   21) "down-after-milliseconds"
   22) "30000"
   23) "info-refresh"
   24) "6165"
   25) "role-reported"
   26) "slave"
   27) "role-reported-time"
   28) "820243"
   29) "master-link-down-time"
   30) "0"
   31) "master-link-status"
   32) "ok"
   33) "master-host"
   34) "192.168.1.7"
   35) "master-port"
   36) "6380"
   37) "slave-priority"
   38) "100"
   39) "slave-repl-offset"
   40) "469296"
2)  1) "name"
    2) "192.168.1.7:6379"
    3) "ip"
    4) "192.168.1.7"
    5) "port"
    6) "6379"
    7) "runid"
    8) "ec173789d1bf22c4a315a8cc66818b83b8302d97"
    9) "flags"
   10) "slave"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "1001"
   19) "last-ping-reply"
   20) "1001"
   21) "down-after-milliseconds"
   22) "30000"
   23) "info-refresh"
   24) "4042"
   25) "role-reported"
   26) "slave"
   27) "role-reported-time"
   28) "75443"
   29) "master-link-down-time"
   30) "0"
   31) "master-link-status"
   32) "ok"
   33) "master-host"
   34) "192.168.1.7"
   35) "master-port"
   36) "6380"
   37) "slave-priority"
   38) "100"
   39) "slave-repl-offset"
   40) "469707"
```

从打印的结果可以看出，故障转移完成之后，旧的master节点只能以从节点的形式存在集群中。

## 工作原理

前面已经介绍并试验了哨兵模式如何搭建以及运行，现在我们来讲讲其原理。实际上哨兵模式是redis的一种工作模式，它主要以监控节点状态以及故障转移为工作目的。哨兵在运行过程中，它以固定的频率去发现节点、故障检测，在主节点故障时以安全的方式去执行故障转移以提高集群的高可用。

### 节点的自动发现

节点发现包括slave节点和哨兵节点。在哨兵的配置文件中，我们只配置了`master`节点的ip和port，而从节点的配置信息我们并没有配置，但是神器的是哨兵节点却知道我们的`slave`节点信息。

#### 从节点发现

哨兵以固定频率向`master`节点发送`INFO`指令，通过该指令就能获取到从节点的信息。如果是第一次执行，哨兵只知道主节点信息，而通过对`master`执行完`INFO`命令后就可以获取到所有的从节点了。因为是周期性的发送`INFO`指令，那么后续有新的从节点加入自然而然的就能发现新节点了。

#### 哨兵节点发现

从节点发现很好理解，但是在配置文件中并没有发现关于其他哨兵节点的配置相关信息，那么哨兵节点是如何发现的呢？

在前面我们知道了哨兵节点通过`INFO`指令获取到了所有主节点和从节点信息，而Redis提供了一种发布订阅的消息通信模式，哨兵就是通过一个约定好的通道`channel`发布和订阅信息就行通信。其整个过程大致如下：

- 每隔固定频率哨兵会通过它所监控的主节点、从节点向`__sentinel__:hello`通道发布一条消息。

- 哨兵会通过它所监控的主节点、从节点向`__sentinel__:hello`通道接收其他哨兵发布的消息。

哨兵发布和接收的消息内容如下：

```
1) "message"
2) "__sentinel__:hello"
3) "192.168.1.7,26380,11d0e0789a3a7eec192f405f9a378fec7f842ed2,1,mymaster,192.168.1.7,6380,1"
```

该消息一共包含了8个信息：哨兵IP、哨兵端口、哨兵runId、当前纪元、主节点名称、主节点IP、主节点端口、主节点纪元。

通过解析这些消息和自己本地的消息进行对比更新，这样哨兵就知道了整个哨兵集群的信息了。

### 监控

通过前面我们知道了节点之间自动发现的原理，接下来介绍一下哨兵是如何监控的。sentinel会以一定频率向`master`和`slave`发送`PING`命令，如果收到该指令的节点未在规定的时间内正确响应哨兵节点，那么哨兵节点就会认为该节点故障了，这种情况我们称之为`主观下线`。很显然主观下线并不能真的认为节点就下线了，因为在网络不好的情况下主观下线的概率还是挺大的，而实际上哨兵认为节点下线是通过`客观下线`的方式来判断。

`客观下线`简单的讲就是一个哨兵说了不算，如果一个哨兵认为一个节点下线了，那么它会询问其他节点哨兵节点当前节点是否下线。如果认为下线的数量超过了指定值，那么就认为该节点真的下线了。

### 故障转移

如果`master`节点被判定为`客观下线`，那么接下来哨兵就会进行故障转移。在故障转移的首要过程中就是选出新的`master`节点。而选择新的master节点会有以下筛选逻辑：

- 从库的在线状态。对于已经下线的从库就不要考虑了，都下线了选上了能有啥用。

- 通过评估`down-after-milliseconds`主观下线的频率来判断网络状态是否稳定。如果经常出现网络问题，那么这个节点也不会考虑。

通过上面两步获取到的节点都是健康的实例，就下来就是在健康的实例中去筛选了。

- slave优先级。可以给不同redis节点设置不同的` slave-priority`值，优先级高的优先选为master。

- 选择数据偏移量差最小的，即slave_repl_offset与 master_repl_offset进度差距，其实就是比较 slave 与 原master 复制进度差距。

- slave runID。在优先级和复制进度都相同的情况下，选用runID最好的，runID越小说明创建时间越早，优先选为master。

通过上面的评判标准选出了新的`master`节点，这时候哨兵会将新的`master`信息通知给到其他`slave`节点，slave节点会修改旧的`REPLICAOF`信息，从新的`master`节点同步数据。同时哨兵还会将新的主节点信息通知给到客户端，这样客户端也获取到集群的最新配置信息。
