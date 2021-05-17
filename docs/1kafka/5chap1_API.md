# **第五节 Kafka API**

## **1、使用 Kafka 原生的 API**

消费者自动提交.

### **1-1 定义自己的生产者：**

```
import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Properties;

/**
 * @ClassName MyKafkaProducer
 * @Description TODO
 * @Author lingxiangxiang
 * @Date 3:37 PM
 * @Version 1.0
 **/
public class MyKafkaProducer {
    private org.apache.kafka.clients.producer.KafkaProducer<Integer, String> producer;

    public MyKafkaProducer() {
        Properties properties = new Properties();
        properties.put("bootstrap.servers", "192.168.51.128:9092,192.168.51.128:9093,192.168.51.128:9094");
        properties.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        // 设置批量发送
        properties.put("batch.size", 16384);
        // 批量发送的等待时间 50ms, 超过 50ms, 不足批量大小也发送
        properties.put("linger.ms", 50);
        this.producer = new org.apache.kafka.clients.producer.KafkaProducer<Integer, String>(properties);
    }

    public boolean sendMsg() {
        boolean result = true;
        try {
            // 正常发送, test2 是 topic, 0 代表的是分区, 1 代表的是 key, hello world 是发送的消息内容
            final ProducerRecord<Integer, String> record = new ProducerRecord<Integer, String>("test2", 0, 1, "hello world");
            producer.send(record);
            // 有回调函数的调用
            producer.send(record, new Callback() {
                @Override
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    System.out.println(recordMetadata.topic());
                    System.out.println(recordMetadata.partition());
                    System.out.println(recordMetadata.offset());
                }
            });
          // 自己定义一个类
            producer.send(record, new MyCallback(record));
        } catch (Exception e) {
            result = false;
        }
        return result;
    }
}
```

定义生产者发送成功的回调函数：

```
import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.RecordMetadata;

/**
 * @ClassName MyCallback
 * @Description TODO
 * @Author lingxiangxiang
 * @Date 3:51 PM
 * @Version 1.0
 **/
public class MyCallback implements Callback {
    private Object msg;

    public MyCallback(Object msg) {
        this.msg = msg;
    }

    @Override
    public void onCompletion(RecordMetadata metadata, Exception e) {
        System.out.println("topic = " + metadata.topic());
        System.out.println("partiton = " + metadata.partition());
        System.out.println("offset = " + metadata.offset());
        System.out.println(msg);
    }
}
```

生产者测试类：在生产者测试类中，自己遇到一个坑，就是最后自己没有加 sleep，就是怎么检查自己的代码都没有问题，但是最后就是没法发送成功消息，最后加了一个 sleep 就可以了。

因为主函数 main 已经执行完退出，但是消息并没有发送完成，需要进行等待一下。当然，你在生产环境中可能不会遇到这样问题，呵呵！

代码如下：

```
import static java.lang.Thread.sleep;

/**
 * @ClassName MyKafkaProducerTest
 * @Description TODO
 * @Author lingxiangxiang
 * @Date 3:46 PM
 * @Version 1.0
 **/
public class MyKafkaProducerTest {
    public static void main(String[] args) throws InterruptedException {
        MyKafkaProducer producer = new MyKafkaProducer();
        boolean result = producer.sendMsg();
        System.out.println("send msg " + result);
        sleep(1000);
    }
}
```

消费者类：

```
import kafka.utils.ShutdownableThread;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Collections;
import java.util.Properties;

/**
 * @ClassName MyKafkaConsumer
 * @Description TODO
 * @Author lingxiangxiang
 * @Date 4:12 PM
 * @Version 1.0
 **/
public class MyKafkaConsumer extends ShutdownableThread {

    private KafkaConsumer<Integer, String> consumer;

    public MyKafkaConsumer() {
        super("KafkaConsumerTest", false);
        Properties properties = new Properties();
        properties.put("bootstrap.servers", "192.168.51.128:9092,192.168.51.128:9093,192.168.51.128:9094");
        properties.put("group.id", "mygroup");
        properties.put("enable.auto.commit", "true");
        properties.put("auto.commit.interval.ms", "1000");
        properties.put("session.timeout.ms", "30000");
        properties.put("heartbeat.interval.ms", "10000");
        properties.put("auto.offset.reset", "earliest");
        properties.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        this.consumer = new KafkaConsumer<Integer, String>(properties);
    }

    @Override
    public void doWork() {
        consumer.subscribe(Arrays.asList("test2"));
        ConsumerRecords<Integer, String>records = consumer.poll(1000);
        for (ConsumerRecord record : records) {
            System.out.println("topic = " + record.topic());
            System.out.println("partition = " + record.partition());
            System.out.println("key = " + record.key());
            System.out.println("value = " + record.value());
        }
    }
}
```

### **1-2 消费者的测试类**

```
/**
 * @ClassName MyConsumerTest
 * @Description TODO
 * @Author lingxiangxiang
 * @Date 4:23 PM
 * @Version 1.0
 **/
public class MyConsumerTest {
    public static void main(String[] args) {
        MyKafkaConsumer consumer = new MyKafkaConsumer();
        consumer.start();
        System.out.println("==================");
    }
}
```

### **1-3 消费者同步手动提交**

前面的消费者都是以自动提交 Offset 的方式对 Broker 中的消息进行消费的，但自动提交 可能会出现消息重复消费的情况。


所以在生产环境下，很多时候需要对 Offset 进行手动提交， 以解决重复消费的问题。


手动提交又可以划分为同步提交、异步提交，同异步联合提交。这些提交方式仅仅是 `doWork()` 方法不相同，其构造器是相同的。

所以下面首先在前面消费者类的基础上进行构造器的修改，然后再分别实现三种不同的提交方式。

同步提交方式是，消费者向 Broker 提交 Offset 后等待 Broker 成功响应。若没有收到响应，则会重新提交，直到获取到响应。


而在这个等待过程中，消费者是阻塞的。其严重影响了消费者的吞吐量。

修改前面的 `MyKafkaConsumer.java`, 主要修改下面的配置：

```
import kafka.utils.ShutdownableThread;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Collections;
import java.util.Properties;

/**
 * @ClassName MyKafkaConsumer
 * @Description TODO
 * @Author lingxiangxiang
 * @Date 4:12 PM
 * @Version 1.0
 **/
public class MyKafkaConsumer extends ShutdownableThread {

    private KafkaConsumer<Integer, String> consumer;

    public MyKafkaConsumer() {
        super("KafkaConsumerTest", false);
        Properties properties = new Properties();
        properties.put("bootstrap.servers", "192.168.51.128:9092,192.168.51.128:9093,192.168.51.128:9094");
        properties.put("group.id", "mygroup");
      // 这里要修改成手动提交
        properties.put("enable.auto.commit", "false");
        // properties.put("auto.commit.interval.ms", "1000");
        properties.put("session.timeout.ms", "30000");
        properties.put("heartbeat.interval.ms", "10000");
        properties.put("auto.offset.reset", "earliest");
        properties.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        this.consumer = new KafkaConsumer<Integer, String>(properties);
    }
    @Override
    public void doWork() {
        consumer.subscribe(Arrays.asList("test2"));
        ConsumerRecords<Integer, String>records = consumer.poll(1000);
        for (ConsumerRecord record : records) {
            System.out.println("topic = " + record.topic());
            System.out.println("partition = " + record.partition());
            System.out.println("key = " + record.key());
            System.out.println("value = " + record.value());

          //手动同步提交
          consumer.commitSync();
        }

    }
}
```

### **1-4 消费者异步手工提交**

手动同步提交方式需要等待 Broker 的成功响应，效率太低，影响消费者的吞吐量。

```
import kafka.utils.ShutdownableThread;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Collections;
import java.util.Properties;

/**
 * @ClassName MyKafkaConsumer
 * @Description TODO
 * @Author lingxiangxiang
 * @Date 4:12 PM
 * @Version 1.0
 **/
public class MyKafkaConsumer extends ShutdownableThread {

    private KafkaConsumer<Integer, String> consumer;

    public MyKafkaConsumer() {
        super("KafkaConsumerTest", false);
        Properties properties = new Properties();
        properties.put("bootstrap.servers", "192.168.51.128:9092,192.168.51.128:9093,192.168.51.128:9094");
        properties.put("group.id", "mygroup");
      // 这里要修改成手动提交
        properties.put("enable.auto.commit", "false");
        // properties.put("auto.commit.interval.ms", "1000");
        properties.put("session.timeout.ms", "30000");
        properties.put("heartbeat.interval.ms", "10000");
        properties.put("auto.offset.reset", "earliest");
        properties.put("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        this.consumer = new KafkaConsumer<Integer, String>(properties);
    }

    @Override
    public void doWork() {
        consumer.subscribe(Arrays.asList("test2"));
        ConsumerRecords<Integer, String>records = consumer.poll(1000);
        for (ConsumerRecord record : records) {
            System.out.println("topic = " + record.topic());
            System.out.println("partition = " + record.partition());
            System.out.println("key = " + record.key());
            System.out.println("value = " + record.value());

          //手动同步提交
          // consumer.commitSync();
          //手动异步提交
          // consumer.commitAsync();
          // 带回调公共的手动异步提交
          consumer.commitAsync((offsets, e) -> {
            if(e != null) {
              System.out.println("提交次数, offsets = " + offsets);
              System.out.println("exception = " + e);
            }
          });
        }
    }
}
```

## **2、Spring Boot 使用 Kafka**

现在大家的开发过程中，很多都用的是 Spring Boot 的项目，直接启动了，如果还是用原生的 API，就是有点 Low 了啊，那 Kafka 是如何和 Spring Boot 进行联合的呢？


Maven 配置：

```
<!-- https://mvnrepository.com/artifact/org.apache.kafka/kafka-clients -->
    <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka-clients</artifactId>
      <version>2.1.1</version>
    </dependency>
```

添加配置文件，在 `application.propertie`s 中加入如下配置信息：

Kafka 连接地址：

```
spring.kafka.bootstrap-servers = 192.168.51.128:9092,10.231.128.96:9093,192.168.51.128:9094
```

生产者：

```
spring.kafka.producer.acks = 0
spring.kafka.producer.key-serializer = org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer = org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.retries = 3
spring.kafka.producer.batch-size = 4096
spring.kafka.producer.buffer-memory = 33554432
spring.kafka.producer.compression-type = gzip
```

消费者：


```
spring.kafka.consumer.group-id = mygroup
spring.kafka.consumer.auto-commit-interval = 5000
spring.kafka.consumer.heartbeat-interval = 3000
spring.kafka.consumer.key-deserializer = org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer = org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.auto-offset-reset = earliest
spring.kafka.consumer.enable-auto-commit = true
# listenner, 标识消费者监听的个数
spring.kafka.listener.concurrency = 8
# topic的名字
kafka.topic1 = topic1
```

生产者：

```
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;

@Service
@Slf4j
public class MyKafkaProducerServiceImpl implements MyKafkaProducerService {
        @Resource
    private KafkaTemplate<String, String> kafkaTemplate;
        // 读取配置文件
    @Value("${kafka.topic1}")
    private String topic;

    @Override
    public void sendKafka() {
      kafkaTemplate.send(topic, "hell world");
    }
}
```

消费者：

```
@Component
@Slf4j
public class MyKafkaConsumer {
  @KafkaListener(topics = "${kafka.topic1}")
    public void listen(ConsumerRecord<?, ?> record) {
        Optional<?> kafkaMessage = Optional.ofNullable(record.value());
        if (kafkaMessage.isPresent()) {
            log.info("----------------- record =" + record);
            log.info("------------------ message =" + kafkaMessage.get());
}
```