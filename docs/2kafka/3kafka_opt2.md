# **L3 kafka应用经验**

## **1、Kafka常用操作指令**

Kafka是一个开源的，分布式的，高吞吐量的消息系统。

kafka对消息保存时根据Topic进行归类，发送消息者成为Producer,消息接受者成为Consumer,此外kafka集群由多个kafka实例组成，每个实例(server)成为broker。

无论是kafka集群，还是producer和consumer都依赖于zookeeper来保证系统可用性，并保存集群的一些meta信息。

本文主要介绍kafka常用操作命令，方便在日常项目维护中使用，熟练使用以下命令，对日常kakfa运维将带来极大的便利。


### **1-1 kafka常用名词介绍**

**Topics（主题）**

属于特定类别的消息流称为主题。数据存储在主题中。Topic相当于Queue。 

主题被拆分成分区。**每个这样的分区包含不可变有序序列的消息**。分区被实现为具有相等大小的一组分段文件。 

**Partition（分区）**

一个Topic可以分成多个Partition，这是为了平行化处理。

每个Partition内部消息有序，其中每个消息都有一个offset序号，消费者根据offset序号来消费数据，已经消费的数据通过offset值记录，不会重复消费。


一个Partition只对应一个Broker，一个Broker可以管理多个Partition。

**Partition offset（分区偏移）** 

每个分区消息具有称为 offset 的唯一序列标识。 

**Replicas of partition（分区备份）**

副本只是一个分区的备份。副本从不读取或写入数据。它们用于防止数据丢失。 

**Kafka Cluster（Kafka集群）** 

Kafka有多个代理被称为Kafka集群。可以扩展Kafka集群，无需停机。这些集群用于管理消息数据的持久性和复制。 

**Producers（生产者）** 

生产者是发送给一个或多个Kafka主题的消息的发布者。生产者向Kafka经纪人发送数据。每当生产者将消息发布给代理时，代理只需将消息附加到最后一个段文件。实际上，该消息将被附加到分区。生产者还可以向他们选择的分区发送消息。 

**Consumers（消费者）** 

Consumers从经纪人处读取数据。消费者订阅一个或多个主题，并通过从代理中提取数据来使用已发布的消息。

Consumer自己维护消费到哪个offet

每个Consumer都有对应的group

group内是queue消费模型：各个Consumer消费不同的partition，因此一个消息在group内只消费一次

group间是`publish-subscribe`消费模型：各个group各自独立消费，互不影响，因此一个消息被每个group消费一次。

## **2、kafka常用命令**

### **2-1 创建topic**

```
./kafka-topics.sh --create --topic topic_name --replication-factor 2 --partitions 10 --zookeeper 127.0.0.1:2181
```

### **2-2 查看所有topic列表**

```
./kafka-topics.sh --zookeeper  127.0.0.1:2181 --list
```

### **2-3 查看指定topic信息**

```
./kafka-topics.sh --zookeeper  127.0.0.1:2181 --describe --topic topic_name 
```

### **2-4 控制台向topic生产数据**

```
./kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic topic_name 
```

### **2-5 控制台消费topic的数据**

```
./kafka-console-consumer.sh --zookeeper  127.0.0.1:2181 --topic topic_name 
```

### **2-6 查看topic某分区偏移量最大（小）值**

```
./kafka-run-class.sh kafka.tools.GetOffsetShell --topic topic_name  --time -1 --broker-list  127.0.0.1:9092 --partitions 0
```

注：time为`-1`时表示最大值，time为`-2`时表示最小值

### **2-7 增加topic分区数**

为`topic_name`  增加到10个分区

```
./kafka-topics.sh --zookeeper  127.0.0.1:2181 --alter --topic topic_name  --partitions 10
```

### **2-8 删除topic**

**慎用**，只会删除zookeeper中的元数据，消息文件须手动删除

```
./kafka-topics.sh --zookeeper  127.0.0.1:2181  --topic topic_name  --delete
```

### **2-9 查看consumer组内消费的offset**

```
./kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181 --group test --topic topic_name 
 ./kafka-consumer-offset-checker.sh --zookeeper  127.0.0.1:2181 --group group1 --topic topic_name 
```

### **2-10 查看kafka某分区日志具体内容**

```
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test3-0/1111111.log  --print-data-log  
```

### **2-11 获取正在消费的topic的group的offset**

```
./kafka-consumer-groups.sh --new-consumer --describe --group test6 --bootstrap-server  127.0.0.1:9092
```

### **2-12 显示消费者**

```
./kafka-consumer-groups.sh --bootstrap-server  127.0.0.1:9092 --list --new-consume
```

### **2-13 消费的topic查看**

```
./bin/kafka-console-consumer.sh --topic topic_name --zookeeper  127.0.0.1:2181  --from-beginning >> /home/123.txt
```

## **3、通过kafka生产者脚本模拟数据发送**

kafka的功能很多，使用也越来越广泛。但是不管是把 Kafka 作为消息、消息队列、总线还是数据存储平台来使用 ，总是需要有一个可以往 Kafka 写入数据的生产者和一个可以从 Kafka读取数据的消费者，或者一个兼具两种角色的应用程序。


在kafka的安装包中，都会提供一个名字为`kafka-console-producer.sh`的脚本，这个脚本使用来模拟生产者向kafka中指定的topic发送数据的，然后我们可以通过`kafka-console-consumer.sh`这个脚本，到kafka中去消费指定的topic接收到的消息。

通过kafka自带的生产者脚本，就可以模拟数据发送到kafka中指定的topic，在验证一些系统功能或者演示系统功能是很有必要的。


**Kafka的生产者有如下三个必选的属性**：

* `bootstrap.servers`，指定broker的地址清单
* `key.serializer`必须是一个实现`org.apache.kafka.common.serialization.Serializer`接口的类，将key序列化成字节数组。注意：`key.serializer` 必须被设置，即使消息中没有指定key
* `value.serializer`，将value序列化成字节数组

**主要使用的命令脚本如下：**

**生产者向kafka中指定的topic发送数据：**


```
./kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic topic_name 
```


通过执行这个脚本，可以模拟数据发送到指定的topic，上述broker-list后面的地址为kafka的ip地址+端口。

输入的数据可以是任何形式的数字或者字符串等，输入完成后回车即可发送出去。

**消费者来消费topic收到的数据：**

```
./kafka-console-consumer.sh --zookeeper  127.0.0.1:2181 --topic topic_name
```

采用kafka自带的消费脚本可以消费指定topic中的数据，如果需要和其他系统对接，可以基于kafka来实现数据的转存、对接。


## **4、Kafka如何删除topic历史数据**


在生产环境中，有一个topic的数据量一般都很多，很多数据如果保存的时间较长，就有可能成为脏数据，那么有没有什么好的办法可以将这部分历史数据删除，并且对保存的数据实现循环覆盖呢？
答案是肯定。

比如有这样的需求：kafka中的topic数据保留48小时即可，超过这个时间的数据不希望保存，这样可以节省磁盘空时占用率。

针对topic中历史数据的删除、定时删除等配置，都可以通过kafka自带的脚本实现。如下列举集中常用的使用场景：


**如何确认kafka中的TOPIC是否有历史数据：**

其实这个比较简单，通过采用kafka自带的脚本可以检测topic中是否有历史数据，将结果重定向到一个文件中即可。

将TOPIC缓存的数据临时重定向到一个文件中，用来临时去除TOPIC的数据。

```
./kafka-console-consumer.sh --zookeeper 127.0.0.1:2181 --topic topic_name --from-beginning  >> /home/1111.txt
```

然后vim 这个文件即可。

**如果要想清除这个topic中的历史数据，可以采用如下办法：**

```
./kafka-topics.sh --zookeeper localhost:2181--alter --topic topic_name --config retention.ms=1000
```

这个语句的意思是将topc的缓存时间修改成1000ms，这样1s钟之前的数据就会被删除，从而实现删除topic历史数据的功能。

**如果要想实现定时删除topic中的数据，如开头说的只保留48小时的数据，超过这个时间的数据不希望保存，可以采用如下办法：**

```
./kafka-topics.sh --zookeeper localhost:2181--alter --topic topic_name --config retention.ms=172800000
```

这个语句的意思是将topc的缓存时间修改成172800000ms，这样48小时之前的数据就会被删除，从而实现kafka中的topic数据仅保存2天，超过2天的数据就会被自动被删除。
