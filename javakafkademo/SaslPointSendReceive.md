

# SASL接入点PLAIN机制收发消息

本文介绍Java客户端如何在VPC环境下通过SASL接入点接入消息队列CKafka版并使用PLAIN机制收发消息。

#### 前提条件

- [安装1.8或以上版本JDK](https://www.oracle.com/java/technologies/javase-downloads.html)
- [安装2.5或以上版本Maven](http://maven.apache.org/download.cgi#)
- [SASL用户授权](https://help.aliyun.com/document_detail/162310.htm#task-2476906)



#### 添加Java依赖库

在pom.xml中添加以下依赖。

```java
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.10.2.2</version>
</dependency>
```

#### 准备配置

- 创建JAAS配置文件ckafka_client_jaas.conf。

  ```
  KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="yourinstance#yourusername"
  password="yourpassword";
  };
  ```

  **说明：**username 是`实例 ID` + `#` + `配置的用户名`，password 是配置的用户密码。

- 创建消息队列CKafka版配置文件kafka.properties。

  ```
  ##SASL接入点，通过控制台获取。
  bootstrap.servers=xxxx
  ##Topic，通过控制台创建。
  topic=xxxx
  ##Consumer Group
  group.id=xxxx
  ##JAAS配置文件。
  java.security.auth.login.config.plain=/xxxx/ckafka_client_jaas.conf
  ```

- 创建配置文件加载程序CKafkaConfigurer.java。

  ```java
  public class CKafkaConfigurer {
  
      private static Properties properties;
  
      public static void configureSaslPlain() {
          //如果用-D或者其它方式设置过，这里不再设置。
          if (null == System.getProperty("java.security.auth.login.config")) {
              //请注意将XXX修改为自己的路径。
              System.setProperty("java.security.auth.login.config",
                      getCKafkaProperties().getProperty("java.security.auth.login.config.plain"));
          }
      }
  
      public synchronized static Properties getCKafkaProperties() {
          if (null != properties) {
              return properties;
          }
          //获取配置文件kafka.properties的内容。
          Properties kafkaProperties = new Properties();
          try {
              kafkaProperties.load(CKafkaProducerDemo.class.getClassLoader().getResourceAsStream("kafka.properties"));
          } catch (Exception e) {
              System.out.println("getCKafkaProperties error");
          }
          properties = kafkaProperties;
          return kafkaProperties;
      }
  }
  
  ```

  

#### 发送消息

1. 创建发送消息程序KafkaSaslProducerDemo.java。

   ```java
   public class KafkaSaslProducerDemo {
   
       public static void main(String args[]) {
           //设置JAAS配置文件的路径。
           CKafkaConfigurer.configureSaslPlain();
   
           //加载kafka.properties。
           Properties kafkaProperties =  CKafkaConfigurer.getCKafkaProperties();
   
           Properties props = new Properties();
           //设置接入点，请通过控制台获取对应Topic的接入点。
           props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaProperties.getProperty("bootstrap.servers"));
   
           //接入协议。
           props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, "SASL_PLAINTEXT");
           //Plain方式。
           props.put(SaslConfigs.SASL_MECHANISM, "PLAIN");
   
           //消息队列Kafka版消息的序列化方式。
           props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
           props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
           //请求的最长等待时间。
           props.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 30 * 1000);
           //设置客户端内部重试次数。
           props.put(ProducerConfig.RETRIES_CONFIG, 5);
           //设置客户端内部重试间隔。
           props.put(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG, 3000);
           //构造Producer对象，注意，该对象是线程安全的，一般来说，一个进程内一个Producer对象即可。
           KafkaProducer<String, String> producer = new KafkaProducer<>(props);
   
           //构造一个消息队列Kafka版消息。
           String topic = kafkaProperties.getProperty("topic"); //消息所属的Topic，请在控制台申请之后，填写在这里。
           String value = "this is ckafka msg value"; //消息的内容。
   
           try {
               //批量获取Future对象可以加快速度。但注意，批量不要太大。
               List<Future<RecordMetadata>> futures = new ArrayList<>(128);
               for (int i =0; i < 100; i++) {
                   //发送消息，并获得一个Future对象。
                   ProducerRecord<String, String> kafkaMessage = new ProducerRecord<>(topic, value + ": " + i);
                   Future<RecordMetadata> metadataFuture = producer.send(kafkaMessage);
                   futures.add(metadataFuture);
   
               }
               producer.flush();
               for (Future<RecordMetadata> future: futures) {
                   //同步获得Future对象的结果。
                       RecordMetadata recordMetadata = future.get();
                       System.out.println("Produce ok:" + recordMetadata.toString());
               }
           } catch (Exception e) {
               //客户端内部重试之后，仍然发送失败，业务要应对此类错误。
               System.out.println("error occurred");
           }
       }
   }
   ```

2. 编译并运行KafkaSaslProducerDemo.java发送消息。

- 运行结果(输出)

```bash
Produce ok:ckafka-topic-demo-0@198
Produce ok:ckafka-topic-demo-0@199

```

#### 消费消息

- 单个Consumer订阅消息， 创建单Consumer订阅消息程序KafkaSaslConsumerDemo.java

  ```java
  public class KafkaSaslConsumerDemo {
      public static void main(String args[]) {
          //设置JAAS配置文件的路径。
          CKafkaConfigurer.configureSaslPlain();
  
          //加载kafka.properties。
          Properties kafkaProperties =  CKafkaConfigurer.getCKafkaProperties();
  
          Properties props = new Properties();
          //设置接入点，请通过控制台获取对应Topic的接入点。
          props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaProperties.getProperty("bootstrap.servers"));
  
          //接入协议。
          props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, "SASL_PLAINTEXT");
          //Plain方式。
          props.put(SaslConfigs.SASL_MECHANISM, "PLAIN");
          //两次Poll之间的最大允许间隔。
          //消费者超过该值没有返回心跳，服务端判断消费者处于非存活状态，服务端将消费者从Consumer Group移除并触发Rebalance，默认30s。
          props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
          //每次Poll的最大数量。
          //注意该值不要改得太大，如果Poll太多数据，而不能在下次Poll之前消费完，则会触发一次负载均衡，产生卡顿。
          props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 30);
          //消息的反序列化方式。
          props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
          props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
          //当前消费实例所属的消费组，请在控制台申请之后填写。
          //属于同一个组的消费实例，会负载消费消息。
          props.put(ConsumerConfig.GROUP_ID_CONFIG, kafkaProperties.getProperty("group.id"));
          //构造消费对象，也即生成一个消费实例。
          KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
          //设置消费组订阅的Topic，可以订阅多个。
          //如果GROUP_ID_CONFIG是一样，则订阅的Topic也建议设置成一样。
          List<String> subscribedTopics =  new ArrayList<String>();
          //如果需要订阅多个Topic，则在这里添加进去即可。
          //每个Topic需要先在控制台进行创建。
          String topicStr = kafkaProperties.getProperty("topic");
          String[] topics = topicStr.split(",");
          for (String topic: topics) {
              subscribedTopics.add(topic.trim());
          }
          consumer.subscribe(subscribedTopics);
  
          //循环消费消息。
          while (true){
              try {
                  ConsumerRecords<String, String> records = consumer.poll(1000);
                  //必须在下次Poll之前消费完这些数据, 且总耗时不得超过SESSION_TIMEOUT_MS_CONFIG。
                  for (ConsumerRecord<String, String> record : records) {
                      System.out.println(String.format("Consume partition:%d offset:%d", record.partition(), record.offset()));
                  }
              } catch (Exception e) {
                  System.out.println("consumer error!");
              }
          }
      }
  }
  ```

- 编译并运行KafkaSaslConsumerDemo.java消费消息。


- 运行结果(输出)

```bash

Consume partition:0 offset:298
Consume partition:0 offset:299

```
  