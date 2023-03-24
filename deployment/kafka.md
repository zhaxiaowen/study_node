# Kafka

### kafka 堆积

**案例 1：**

**故障现象：**

1.kafka 消费堆积，消费组成员数量一直变化，一直有实例离开、加入消费组，触发 rebalance

2.10 个分区，只消费 1 个分区，其他分区不消费

**腾讯云专家给出的建议：**

1. 修改 records 参数

2. 消费组的实例数量减少到 1，看下是否有改善

3. 重启有问题的节点

**故障处理：**

1.broker 正常，该 topic 其他消费组都正常

2. 修改 max.poll.interval.ms 心跳上报时间，调长

3. 修改 max.poll.records 批处理的条数，调小，调成 100；如果在心跳上报时间内批量处理的条数没有处理完成，kafka 会认为客户端不行，会触发 rebalance, 并重复消费

4. 重启所有节点，没有再出现 rebalance, 恢复正常消费

**故障原因：** 可能是部分应用有问题导致

案例 2：

故障现象：kafka 堆积

故障原因：

1. 直接原因：启动灾备节点触发 rebalance groupid, 消费组被分配到灾备节点，灾备节点消费速度慢导致堆积

2. 根本原因：该 topic 为保证顺序消费，为单分区，消费者只有 1 个线程可以消费，同时线程需要处理多次远程调用业务逻辑，灾备节点网络延迟 2.5ms 放大调用处理时间

处理方法：

1. 下掉灾备节点流量

2. 因为无法增加消费者，所以对消费者服务器做升配处理，升配后，消费能力有所提高，等待消费完成

影响：影响货物分拣效率，影响货物时效

原因：灾备节点启动，kafka 消费者实例增加触发重平衡，灾备消费能力弱导致堆积

kafka topic 消息堆积

影响：暂未收到业务方反馈异常，目前消息堆积数 280w+，具体影响待评估

原因：通过调整业务线配置参数的批处理条数，由 1000 减少到 100，重启服务后恢复。

### kafka 节点扩容

**原本节点：**0,1,2,3,4，共 5 个节点

**扩容后：**0,1,2,3,4,5，共 6 个节点

**注意：** 扩容需要注意打通应用到扩容节点的网络权限，应用配置文件可以不用更改。应用只要连接到一个节点获取到元数据后，就可以获取到所有节点。

**1、新节点启动加入集群**

1.1、启动新扩容节点

1.2、查看启动 kafka 启动日志

1.3、查看机器新节点是否加入

\# /usr/local/kafka/bin/kafka-topics.sh --zookeeper IP:2181 --describe

**2、数据均衡**

2.1 找出大分区

2.2 规划迁移操作

将大分区迁移到新增节点，减少老节点磁盘容量压力。

2.3 准备 expand-cluster-reassignment-action.json 文件，并检查是否为 json 格式

\# cat expand-cluster-reassignment-action.json

{

  "version": 1,

  "partitions": [

​    {

​      "topic": "test-device",

​      "partition": 1,

​      "replicas": [

​        0,5

​      ],

​      "log_dirs": [

​        "any","any"

​      ]

​    },

​    {

​      "topic": "test-device",

​      "partition": 2,

​      "replicas": [

​        2, 5

​      ],

​      "log_dirs": [

​        "any", "any"

​      ]

​    }

  ]

}

\# jq . expand-cluster-reassignment-action.json

2.6 通过 expand-cluster-reassignment-action.json 执行迁移动作 [限流 30MB/s]

执行迁移操作

\# /usr/local/kafka/bin/kafka-reassign-partitions.sh --zookeeper  IP:2181 --reassignment-json-file expand-cluster-reassignment-action.json -throttle 30000000 --execute

2.7 查看动作

\# /usr/local/kafka/bin/kafka-reassign-partitions.sh --zookeeper  IP:2181 --reassignment-json-file expand-cluster-reassignment-action.json --verify

**3、验证**

3.1 使用 kafkatools 查看在线 broker，或者 zookeeper 里面看

3.2 均衡验证

查看均衡情况

\# /usr/local/kafka/bin/kafka-reassign-partitions.sh --zookeeper  IP:2181 --reassignment-json-file expand-cluster-reassignment-action.json --verify

查看当前 topic ISR 列表，是否解除限流

\# /usr/local/kafka/bin/kafka-topics.sh --zookeeper IP:2181 --describe

**经验：**

（1）打开 json 文件可以知道是以分区为单位迁移的，比如迁移某个 Topic 的 0 号分区，那么就是把它的副本重新分布位置

（2）建议先迁移一个进行验证

（3）限速建议 15M/s

（4）观察 CPU、内存和磁盘

（5）需要提前迁移后磁盘的占用情况，评估磁盘是否够用

（6）先迁移到大分区到新节点（其实只建议迁移大的分区到新节点，不要轻易用脚本生产所有分区然后迁移）

（7）只迁有必要移动的分区，没有必要的不要迁，注意限速

### Kafka 分区 leader 迁移案例

首先，写这篇文章的目的是为了帮助在 kafka 分区迁移 leader 可能遇到和我一样的问题的同学，同时也为了记录自己迁移遇到的问题。

​	我们所用的生产环境的 kafka 版本为 0.11.1，一开始只有 7 台机器作为集群，后来又加入了新的 7 台机器配置和原集群一样，新加集群是因为原来的 7 台机器的 cpu 到达了 80%，这样总集群达到了 14 台。由于新加机器 kafka 不会自动平衡，需要自己进行分区 leader 的迁移。我们一开始选取了 3 种方案：

**一**、**降低数据保留时间，精准迁移** 

大致思路为：

1. 找出需要操作的 Topic
2. 调整这些 Topic 的数据保留时间

3. 对这些 Topic 进行迁移
4. 等待迁移完成，完成 Leader 切换，均分集群压力。

**二、往指定节点上添加分区，均分压力**

大致思路为：

1. 找出需要操作的 Topic
2. 评估这些 Topic 需要导多少流量到新节点。这一步比较好做，一般比如要把 30%，50% 的流量导入到新节点，大概评估一下即可。

3. 扩容目标 Topic 的分区数，将这些新增的分区指定扩容到新的节点上。

**三：切换 Leader，降低单机负载**

大致思路为：

1. 找出负载高的节点上的所有 Leader
2. 将节点上分区的 Leader 切换到负载较低的 Follower 节点上

​	最后，我们选用了第三种方法，原因是第一二种与业务沟通下来不可进行修改。接下来按照官方推荐的方法为：

1、执行 kafka-reassign-partitions.sh --generate 生成迁移的 json 文件，这步我们没有进行操作，是因为有现成的 api 能自动生成执行 的 json，就是因为这一步没有做，导致了我们生成的 json 与推荐的不一致，推荐的为：你指定--broker-list，那么他会把你 topic 分区都迁移到你指定的 broker 上，例如原来 topic 为 3 副本，partition-0 的 replica 为{0,1,2}，指定 broker-list 为“4,5,6”，那么就会生成一个{4,5,6}的 replica，而我们的 api 生成的副本为{4,1.2}, 就是随机选取一个新的 brolerid 放入第一个位置。保留原来的两个副本。

​	这么做的原因是因为：害怕把分区都移到新的上面，迁移的这段时间可能导致不能进行读写。然后就是执行 kafka-reassign-partitions.sh --throttle --execute ，这一步又遇到了问题，因为内部之前自己封了一下 kafka admin 的接口，可以通过页面直接进行执行 kafka-reassign-partitions.sh --throttle --execute 的命令，可能之前的人理解的不到位，只是执行了 kafka-reassign-partitions.sh --throttle --execute 没有执行 kafka-reassign-partitions.sh --verify 导致的问题就是当你执行命令：kafka-topics.sh --describe --zookeeper --topic xxx 你会发现除了自己修改的配置外，还会有两个配置在其中：leader.replication.throttled.replicas 与 follower.replication.throttled.replicas，

​	正常情况下这两个配置默认是空的，执行的时候不会出现此配置，但是我们的确确实实出现了，这两项是 leader 与 follower 之间加了限速，那么就意味着不正常，可是跑了这么长时间了为啥没有出现流量的异常，我就很匪夷所思。通过查找 zk 所知：在分区 leader 与 follower 的限速上。一共有 4 条配置：

leader.replication.throttled.rate，follower.replication.throttled.rate，leader.replication.throttled.replicas

follower.replication.throttled.replicas，

​	其中前两条：leader.replication.throttled.rate，follower.replication.throttled.rate 是加在 broker 上的，通过在 zk 中 get /configs/brokers/{brokerid}所获得，后两条：leader.replication.throttled.replicas，follower.replication.throttled.replicas 是在 topic 上的配置，通过 get /config/topics/{topicname}获得，上面说的流量没有异常的原因是：我在 get /configs/brokers/{brokerid}后

leader.replication.throttled.rate，follower.replication.throttled.rate 是空的，那么就没有此配置，也就是说 broker 侧的没有加上限速的配置，还是以默认的最大速率进行同步，topic 侧显示了限速的副本，对应的配置：leader.replication.throttled.replicas，follower.replication.throttled.replicas

​	那么，通过执行命令：kafka-configs.sh --zookeeper xxx --entity-type topics --entity-name xxx --alter --delete-config follower.replication.throttled.replicas, leader.replication.throttled.replicas 把 topic 配置删除，实现配置的还原。

​	最后，如果有同学需要进行副本的迁移，那么一定要运行--verfy 命令当出现：Throttle was removed. 才证明完成了限速也解除了，leader 的迁移时，原来的 leader 是能进行读写的，所以不需要像我一样还放两个副本在旧机器上
