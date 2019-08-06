---
title: Kafka系列2——Producer和Consumer
date: 2019-08-02 14:16:45
categories: kafka
tags: kafka
---

##1. Kafka Producer.
1.1 简介.
>每个producer是独立工作的，与其他producer实例间没有关联。因为上一节中我们提过，Kafka的消息是三元值，即topic-partition-message.因此，
producer向某个topic发送消息，首先需要确定哪个分区，这就是分区器作用。（自定义或系统自带）.kafka默认分区器，如果消息指定key，则根据key
的哈希值选择目标分区，如果没有指定key，则使用轮询方式去欸的那个目标分区.确定分区后，producer要寻找这个分区对应的leader，也就是该分区
leader副本所在的Kafka broker.整个producer的工作流程如下图：
![](/images/Kafka_1.png)

1.2 Java版本的producer实例.
```
public class ProducerTest {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put("bootstrap.servers","devhost30:9092");
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("acks", "-1");
        properties.put("retries", 2);
        properties.put("batch.size", 323840);
        properties.put("linger.ms", 10);
        properties.put("buffer.memory", 33554432);
        properties.put("max.block.ms", 3000);
        properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "ank.hao.producer.partition.AnkPartitioner");
        properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, "ank.hao.producer.interceptor.AnkInterceptor");

        Producer<String,String> producer = new KafkaProducer<String, String>(properties);
//        for(int i=0;i<100;i++){
//            producer.send(new ProducerRecord<String, String>("first-topic", Integer.toString(i), "message-"+Integer.toString(i)));
//        }
        for(int m=10;m<13;m++){
            producer.send(new ProducerRecord<String, String>("first-topic", Integer.toString(m), "callback-" + Integer.toString(m)), new Callback() {
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    if(e == null){
                        //消息发送成功
                        System.out.println("发送成功了喵："+recordMetadata.topic());
                    }else {
                        if(e instanceof RetriableException){
                            //处理可重试瞬时异常
                        } else {
                            //处理不可重试异常
                        }
                    }
                }
            });
        }
//        for(int m=20;m<3;m++){
//            try {
//                producer.send(new ProducerRecord<String, String>("first-topic", Integer.toString(m), "sync-" + Integer.toString(m))).get();
//            } catch (InterruptedException e) {
//                e.printStackTrace();
//            } catch (ExecutionException e) {
//                e.printStackTrace();
//            }
//        }
        producer.close();
    }
}
```
>上面示例代码中，使用了三种发送消息方式：fire and forget、异步发送、同步发送；producer的参数可在官网查询详情，其中需要 注意的是,前三个参数必须指定，
且bootstrap.servers需要采用主机名方式配置，不可用IP地址；其他重要参数：
>* acks. 用于控制producer消息的持久性.相当于broker反馈的持久化ISR数量.0表示完全不管反馈结果继续下一个消息发送；all或-1表示会等待ISR中所有副本成功
写入各自本地的日志后才继续下一个；1表示leader broker将消息成功写入本地日志后继续下一个；
>* buffer.memory. 指定producer用于缓存消息的缓冲区大小，单位字节.默认32M.
>* compression.type. 设置producer端是否压缩消息，默认none，不压缩.
>* retries. 失败重试次数，Kafka broker在处理写入请求时可能因为可恢复的瞬时故障（如瞬时leader选举或网络抖动）导致消息发送失败；配合使用的几个参数：
retry.backoff.ms，重试停顿时间，默认100ms.重试可能造成的问题：消息的重复发送和乱序.
>* batch.size. 每次批量发送消息的大小，单位字节，默认16KB.
>* linger.ms 和batch.size配合使用，每次发送时间延迟.默认0，即无须关心batch是否已被填满.
>* max.request.size 控制producer端能够发送的最大消息大小，默认1MB.
>* request.timeout.ms producer发送请求给broker后等待超时时间，默认30s.

1.3 分区策略.
>Producer分区接口org.apache.kafka.clients.producer.Partitioner，默认的DefaultPartitioner机制如之前所述；如果要自定义分区策略，需要以下步骤：
>* 在procuer端创建一个实现Partitioner的类，主要分区策略在partition()中实现.
>* 在构造KafkaProducer中设置partitioner.class参数.

1.4 消息序列化.
>序列化接口org.apache.kafka.common.serialization.Serializer<T>，默认提供十几种序列器，如StringSerializer、LongSerializer、BytesSerializer等.
自定义序列化步骤如下：
>* 定义数据对象格式.
>* 创建一个实现Serializer<T>的类，在serializer()中实现序列化逻辑.
>* 在构造KafkaProducer中设置key.serializer和value.serializer.

1.5 拦截器.
> producer和consumer都有自己的拦截器链。producer拦截器使用户在消息发送前以及producer回调逻辑前有机会对消息做定制化需求.
自定义拦截器步骤如下：
>* 实现org.apache.kafka.common.Configurable.ProducerInterceptor<K, V>接口；
>* 在构造KafkaProducer中设置interceptor.classes.

1.6 无消息丢失配置.
> 其中一种方式是同步方式发送消息，但效率太低，一般使用下面方式配置.
>producer端配置：
```
  acks=all  //最强程度持久化保证.
  retries=Integer.MAX_VALUE
  max.in.flight.requests.per.connection=1  限制producer在单个broker连接上能够发送的未响应请求数量.
  使用带有回调的send(record,callback)
  Callback逻辑中显示立即关闭producer.
```
>broker端配置：
```
  unclean.leader.election.enable=false 关闭unclean leader选举，即不允许ISR中副本被选举为leader，从而避免broker端因日志水位截断造成消息丢失.
  replication.factor >=3  副本数量
  min.insync.replicas >1  控制消息至少被写入到ISR中多少副本才算成功.
  确保replication.factor > min.insync.replicas 
```

1.7 消息压缩.
> kafka支持的压缩算法：none, gzip, snappy, lz4, zstd.

1.8 多线程处理.
> 有两种基本使用用法：多线程蛋KafkaProducer实例，多线程多KafkaProducer实例.推荐用法：对分区不多的Kafka集群使用第一种，对拥有超多分区集群用第二种.

##2. Kafka Consumer.
2.1 简介.
> 消费者组概念：消费者使用一个group.id标记自己，topic的每条消息都只会被发送到每个订阅它的消费者组的一个消费者实例上.Kafka实现两种模型：
>* 所有consumer实例都属于同一个group——实现基于队列的模型，每条消息只会被一个consumer实例处理.
>* consumer实例都属于不同group——实现基于发布/订阅的模型。每条消息会被所有consumer实例处理.
![](/images/Kafka_2.png)

2.2 Java版本的Consumer实例.
```
public class ConsumerTest {
    public static void main(String[] args) {
        String topic = "first-topic";
        String groupId = "first-group";

        Properties properties = new Properties();
        properties.put(BOOTSTRAP_SERVERS_CONFIG, "devhost30:9092");
        properties.put(GROUP_ID_CONFIG, groupId);
        properties.put(ENABLE_AUTO_COMMIT_CONFIG, "true");
        properties.put(AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        properties.put(AUTO_OFFSET_RESET_CONFIG, "earliest");
        properties.put(KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put(VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);
        consumer.subscribe(Arrays.asList(topic));
        try {
            while (true){
                ConsumerRecords<String,String> records = consumer.poll(Duration.ofSeconds(1));
                for(ConsumerRecord<String,String> record:records){
                    System.out.printf("offset=%d, key=%s, value=%s%n", record.offset(), record.key(), record.value());
                    System.out.println();
                }
            }
        } finally {
            consumer.close();
        }
    }
}
```
>构造一个java.util.Properties对象，至少指定bootstrap.servers、key.deserializer、value.deserializer和group.id的值，同样，bootstrap.servers
使用主机名，不可用IP地址；一些重要参数：
>* session.timout.ms  coordinator监测失败的时间，默认10s
>* max.poll.interval.ms  consumer处理逻辑最大时间.
>* auto.offset.reset  指定无位移信息或位移越界时Kafka的应对策略.
>* enable.auto.commit  指定consumer是否自动提交位移.
>* fetch.max.bytes  指定consumer端单词获取数据的最大字节数
>* max.poll.records  单次poll调用返回的最大消息数
>* heartbeat.interval.ms  rebalance心跳时间间隔
>* connections.max.idle.ms  空闲连接时间，默认9分钟

2.3 订阅topic
>consumer.subscribe(..)订阅topic列表，也可以基于正则表达式订阅topic；consumer订阅是延迟生效的，订阅信息在下次poll调用时才会正式生效；

2.4 消息轮询
>poll内部原理：采用类似于linux IO模型的poll或select等，使用一个线程同时管理多个socket连接，即同时与多个broker通信实现消息的并行读取；
一旦consumer订阅了topic，所有的消费逻辑包括coordinator的协调、消费者组的rebalance以及数据获取都会在主逻辑poll方法的以此调用中执行.
poll方法返回的任一条件：获取足够多可用数据，等待时间超过设定的超时时间.

2.5 位移管理
> 位移： 每个consumer实例都会为它消费的分区维护属于自己的位置信息来记录当前消费信息进度，被称为位移；<br>
> 位移提交： consumer客户端需要定期向kafka集群汇报自己消费数据进度，这一过程被称为位移提交；<br>
> __consumer_offsets: 位移提交会提交到kafka一个内部topic上，这个topic就是__consumer_offsets；数据格式：可以理解为一个KV格式的消息，
key是一个三元组：group.id+topic+分区号，value是offset的指；<br>
> 位移管理： consumer会在kafka集群的所有broker中选择一个作为consumer group的协调者(coordinator)，用于实现组成员管理、消费分配方案
制定以及提交位移等。提交位移主要机制是通过向所属的coordinator发送位移提交请求来实现的；
> 自动提交和手动提交： 默认自动提交，auto.commit.interval.ms可以控制自动提交间隔；手动提交就是用户自行去欸的那个消息何时被真正处理完
并提交位移，设置enable.auto.commit=false，然后调用commitSync或commitAsync；

2.6 rebalance
>rebalance本质上是一组协议，规定了一个consumer group是如何达成一致来分配订阅topic的所有分区的；rebalance触发条件有三个：组成员发生变更，
组订阅topic数发生变更，组订阅topic的分区数发生变更；

2.7 解序列化
>consumer端的解序列化和producer端的序列化是互逆操作，同理可自定义解序列化类；

2.8 多线程消费实例
>* 每个线程维护一个KafkaConsumer实例
>* 单KafkaConsumer实例+多worker线程

2.9 独立consumer
> consumer不仅可以以consumer group形式出现，也可以独立使用，彼此独立工作互不干扰。以下情况适合使用独立consumer：
>* 进程自己维护分区状态，可以固定消费某些分区不用担心消费状态丢失；
>* 进程本身已经是高可用且能够自动重启恢复错误（比如yarn和mesos等容器调度框架），不需要kafka来错误检测和状态恢复；