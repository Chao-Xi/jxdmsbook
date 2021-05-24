# **L2 Kafka实战—常见运维操作**

## **1、集群操作**

### **1-1 启动集群**

```
kafka-server-start.sh -daemon  /usr/local/kafka/config/server.properties
```

### **1-2 停止集群**

```
kafka-server-stop.sh /usr/local/kafka/config/server.properties
```

## **2、Topic操作**

### **1-1 创建Topic**

```
kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 3 --partitions 3 --topic wolf
```

注意：副本数不可大于broker的数量。

Replication factor: 6 larger than available brokers: 3.


### **1-2 删除Topic**

```
kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --delete --topic wolf
```

注意：此处的删除操作只是将该topic标记为删除，并未真正的删除，且要创建一个同名的topic也不会成功。

```
Error while executing topic command : Topic 'wolf' already exists
```

如果打算删除重新创建，可以先修改  `kafka/config/server.properties` ，

在文件的最后加入配置  `delete.topic.enable=true`

则此时执行删除命令将直接删除

### **1-3 修改Topic**

* 增加分区数

```
kafka-topics.sh --bootstrap-server 127.0.0.1:2181 --alter --topic my_topic_name --partitions 40
```

* 修改配置

```
kafka-configs.sh --bootstrap-server 127.0.0.1:9092 --entity-type topics --entity-name my_topic_name --alter --add-config x=y
```

* 修改过期时间

全局配置`server.properties`

```
log.retention.hours=72
log.cleanup.policy=delete
```

* 单独配置

```
kafka-configs.sh --bootstrap-server 127.0.0.1:9092 --alter --entity-name wolf --entity-type topics --add-config retention.ms=86400000
```

* 修改副本因子

首先创建一个json文件，指定相关的配置

```

vim custom_replication.json

{
  "partitions": [{
      "topic": "wolf",
      "partition": 0,
      "replicas": [1]
    },
    {
      "topic": "wolf",
      "partition": 1,
      "replicas": [4]
    },
    {
      "topic": "wolf",
      "partition": 2,
      "replicas":[2]
    }
  ],
  "version": 1
}
```


注意：

* partition：分片的编号
* replicas：指定分片所分布broker的，broker id列表

执行修改副本

```
kafka-reassign-partitions.sh --bootstrap-server 127.0.0.1:9092 --reassignment-json-file /Users/zhaoqiang/Downloads/aa.json --execute 

//以不超多500M/s的速度进行数据迁移【此处的单位是B/s】
kafka-reassign-partitions.sh --bootstrap-server 127.0.0.1:9092 --reassignment-json-file /Users/zhaoqiang/Downloads/aa.json --execute --throttle 50000000

//--verify 用于分区分配的状态
```

### **1-4 查询Topic**

* 罗列所有Topic

```
kafka-topics.sh --list --zookeeper 127.0.0.1:2181
```

* 查看具体topic详情【其中的数字是brokerId】

```
kafka-topics.sh --zookeeper 127.0.0.1:2181 --describe --topic test
```

* 列出与集群的默认配置不同的topic

```
kafka-topics.sh --zookeeper 127.0.0.1:2181 --describe --topics-with-overrides
```

* 列出包含不同步副本的topic

```
kafka-topics.sh --zookeeper 127.0.0.1:2181  --describe --under-replicated-partitions
```

* 列出leader不可用的副本

```
kafka-topics.sh --zookeeper 127.0.0.1:2181  --describe --unavailable-partitions
```

* 修改topic分区数

```
kafka-topics.sh --alter --zookeeper 127.0.0.1:2181  --partitions 4 --topic wolf
```

* 查看选举失败的Topic 分区

```
kafka-topics.sh --zookeeper 127.0.0.1:2181 --describe|grep "Leader: -1"
```

## **3、生产者操作**

### **3-1 生产消息**

```
kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic fusion_center_monitor_metric
```

## **4、消费者操作**

> 注意：旧版本的消费者组信息存储在zookeeper上【--zookeeper】
     新版本的消费者组信息存储在broker上【--bootstrap-server】
     
### **4-1  查看消息**

* 查看指定消费者分组消费过指定topic的消息【实时数据】

```
kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic fox
```

* 查看指定消费者分组消费过指定topic的消息【从第一条数据开始】

```
kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic log_mryx_intelligent_promotion --from-beginning --group wolf
```

### **4-2 查看消息进度**

```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092  --describe --all-groups
```

将消费者组的偏移量导出到 offsets.txt

```
kafka-run-class.sh kafka.tools.ExportZkOffsets --zkconnect 127.0.0.1:2181--group gid --output-file offsets.txt
```

### **4-3 消费组操作**

* 查看所有的消费者组


```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --list
```

* 消费者组描述

```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --group gid
```

* 消费者组中所有活跃成员的列表

```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --group gid --members

kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --group gid --members --all-groups
```

* 消费者组中所有活跃成员及成员所对应的分区列表

```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --all-groups --members --verbose
```

* 消费者组状态

```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --all-groups --state
```

* 删除消费者组

```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --delete --group gid
```

* 重置消费者组偏移量

```
kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --reset-offsets --group consumergroup1 --topic topic1 --to-latest
```

```
--reset-offsets  
 
 需指定Topic --all-topics或--topic
  
  执行选项
  --（默认）以显示要重置的偏移量。
  --execute：执行--reset-offsets过程。
  --export：将结果导出为CSV格式。
  执行方案
--to-datetime <String：datetime>：将偏移量重置为与datetime的偏移量。格式：“ YYYY-MM-DDTHH：mm：SS.sss”
--to-earliest：将偏移量重置为最早的偏移量。
--to-latest：将偏移量重置为最新偏移量。
--shift-by <Long: number-of-offsets>：重置偏移，将当前偏移偏移“ n”，其中“ n”可以为正或负。--from-file：将偏移量重置为CSV文件中定义的值。
--to-current：将偏移量重置为当前偏移量。
--by-duration <String：duration>：将偏移量重置为从当前时间戳记的持续时间偏移量。格式：“ PnDTnHnMnS”
--to-offset：将偏移量重置为特定偏移量。请注意，超出范围的偏移量将调整为可用的偏移量结束。例如，如果偏移量结束为10，偏移量请求为15，则实际上将选择偏移量为10
```