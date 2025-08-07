---
title: SpringCloud集成并使用kafka
date: 2025-08-07 11:45:47
tags: [SpringCloud, Kafka, SpringBoot]
---

最近项目中在做证书签名，需要使用kafka的方式进行数据交换，因为项目中使用了SpringCloud,这里看哈如何集成Kafka。

因为是两个项目进行数据交换，其中一个需要有producer功能，其中一个需要有consumer功能，同时需要保证生产者消费者的traceId是同一个。

### 添加依赖

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>2.9.13</version>
</dependency>
```

### 添加证书

因为连接kafka的方式需要通过证书，所以我把证书文件放到了resources/client-certs文件夹下面

```
resources\client-certs
├─dev
│      kafka.client.truststore.jks
│
├─pp
│      kafka.client.truststore.jks
│
├─prod
│      kafka.client.truststore.jks
│
└─uat
        kafka.client.truststore.jks
```

同样的需要在pom文件中把这个证书文件在打包的时候打进去

#### 指定profile

```xml
	<profiles>
        <profile>
            <id>dev</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <spring.profiles.active>dev</spring.profiles.active>
            </properties>
        </profile>
        <profile>
            <id>uat</id>
            <properties>
                <spring.profiles.active>uat</spring.profiles.active>
                <bvpro.api.version>1.9.0</bvpro.api.version>
            </properties>
        </profile>
        <profile>
            <id>pp</id>
            <properties>
                <spring.profiles.active>pp</spring.profiles.active>
            </properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <spring.profiles.active>prod</spring.profiles.active>
            </properties>
        </profile>
   </profiles>
```

#### build的时候打包

```xml
<build>
        <finalName>${project.artifactId}</finalName>
        <plugins>
            <plugin>
                <artifactId>maven-resources-plugin</artifactId>
                <version>3.2.0</version>
                <executions>
                    <execution>
                        <id>copy-resources</id>
                        <phase>package</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${project.build.directory}/</outputDirectory>
                            <resources>
                                <resource>
                                    <directory>src/main/resources/client-certs/${spring.profiles.active}</directory>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
        <resources>
            <resource>
                <!-- 指定配置文件所在的resource目录 -->
                <directory>src/main/resources</directory>
                <includes>
                    <!-- 选择要打包的配置文件和日志，当然也可以不加打包所有配置文件，然后由具体服务器来选择使用哪个 -->
                    <include>bootstrap.yml</include>
                </includes>
                <filtering>true</filtering>
            </resource>
            <resource>
                <!-- 指定配置文件所在的resource目录 -->
                <directory>src/main/resources</directory>
                <includes>
                    <include>client-certs/**/*.*</include>
                </includes>
            </resource>
        </resources>
    </build>
```
#### 修改dockerfile

因为项目还是使用docker打包的方式运行在aws上的，所以这里还需要修改dockerfile


```dockerfile
# 基础镜像
FROM 891377395322.dkr.ecr.ap-southeast-1.amazonaws.com/ecr-cgn-mut-np-amazoncorretto:11-all


# author
MAINTAINER think123

# 挂载目录
VOLUME /home/think123
# 创建目录
RUN mkdir -p /home/think123
# 指定路径
WORKDIR /home/think123

COPY kafka.client.truststore.jks /home/think123/kafka.client.truststore.jks

# 复制jar文件到路径
COPY ./think123.jar /home/think123/think123.jar
ENV JAVA_OPTS="-Xms516m -Xmx3072m -XX:+UseConcMarkSweepGC"
# 启动认证服务
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar think123.jar"]

```

以上的配置无论你是只想集成消费者还是只想集成生产者，都是必不可少的配置。

### 生产者配置

```yaml
# spring配置
spring:
  kafka:
    bootstrap-servers: server1:9096,server2:9096
    ssl:
      trust-store-location: file:/home/think123/kafka.client.truststore.jks
      trust-store-password: think123
    properties:
      retry:
        backoff:
          ms: 300000    
      sasl:
        mechanism: SCRAM-SHA-512
        jaas:
          config: org.apache.kafka.common.security.scram.ScramLoginModule required username="think123-admin" password="K$|5L59&A!DE234-.a";
      security:
        protocol: SASL_SSL

```
### 消费者配置

```yaml
# spring配置
spring:
  kafka:
    bootstrap-servers: server1:9096,server2:9096
    ssl:
      trust-store-location: file:/home/think123/kafka.client.truststore.jks
      trust-store-password: changeit
    consumer:
      group-id: osc-signature-message-uat
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      auto-offset-reset: latest
      enable-auto-commit: false
      auto-commit-interval: 100ms
      max-poll-records: 50
      properties:
        session:
          timeout:
            ms: 120000
        request:
          timeout:
            ms: 120000
    properties:
      retry:
        backoff:
          ms: 300000    
      sasl:
        mechanism: SCRAM-SHA-512
        jaas:
          config: org.apache.kafka.common.security.scram.ScramLoginModule required username="think123-admin" password="K$|5L59&A!DE234-.a";
      security:
        protocol: SASL_SSL
    listener:
      # 手动确认
      ack-mode: manual_immediate
      # 并发为3,一个group存在3个消费者
      concurrency: 3

```

### 发送消息以及消费消息


启动kafka的时候需要添加`@EnableKafka`注解

#### 配置traceId

```java
@Configuration
@EnableKafka
@Slf4j
public class KafkaConfig {

   public static final String TRACE_ID = "traceId";

    @Bean
    public KafkaTemplate<String, String> tracingProducerFactory(ProducerFactory<String, String> producerFactory) {
        Map<String, Object> producerConfig = new HashMap<>();
        producerConfig.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, TraceIdProducerInterceptor.class.getName());
        producerFactory.updateConfigs(producerConfig);
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public DefaultKafkaConsumerFactoryCustomizer kafkaConsumerFactoryCustomizer() {
        return consumerFactory -> {
            Map<String, Object> props = new HashMap<>(
                    consumerFactory.getConfigurationProperties());

            // 添加拦截器配置
            String interceptors = (String) props.get(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG);
            String interceptorClass = TracingConsumerInterceptor.class.getName();

            if (interceptors == null) {
                props.put(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptorClass);
            } else if (!interceptors.contains(interceptorClass)) {
                props.put(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG,
                        interceptors + "," + interceptorClass);
            }
            // 更新配置
            consumerFactory.updateConfigs(props);
        };
    }

    public static class TraceIdProducerInterceptor implements ProducerInterceptor<String, String> {
        @Override
        public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
            String traceId = LogUtil.traceId();
            if (traceId != null) {
            	// 将traceId放到header里面
                record.headers().add(TRACE_ID, traceId.getBytes());
            }
            return record;
        }

       // 省略其他空方法
    }

    public static class TracingConsumerInterceptor<K, V> implements ConsumerInterceptor<K, V> {

        @Override
        public ConsumerRecords<K, V> onConsume(ConsumerRecords<K, V> records) {
            records.forEach(record -> {
                // 从消息头中提取traceId,放到MDC中
                String traceId = extractTraceIdFromHeader(record);
                if (StringUtils.isNotEmpty(traceId)) {
                    MDC.put(TRACE_ID, traceId);
                }
            });
            return records;
        }

        private String extractTraceIdFromHeader(ConsumerRecord<?, ?> record) {
            Header header = record.headers().lastHeader(TRACE_ID);
            return header != null ? new String(header.value()) : null;
        }
        // 省略其他空方法
    }
}

```
#### 生产者

如果你只需要生产者，那么在你的业务类里面注入KafkaTemplate就行了，然后调用`kafkaTemplate.send()`方法即可

```java
   @Resource
   private KafkaTemplate<String, String> kafkaTemplate;
```

#### 消费者

```java

@Component
@RequiredArgsConstructor
@Slf4j
public class SignatureMessageConsumer {

 	private final ExecutorService signatureHandlerExecutor;

    @KafkaListener(id = "digital-signature-consumer",
            topics = {"${application.kafka.signature-topic:OSC_OSC_DIGITAL_SIGN}"})
    public void signatureMessageConsumer(@Payload String payload, @Header(KafkaHeaders.GROUP_ID) String groupId,
                                         @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                                         @Header(KafkaHeaders.RECEIVED_PARTITION) Integer partition,
                                         @Header(KafkaHeaders.OFFSET) Long offset, Acknowledgment ack) throws Exception {
        log.info("Received msg of signature: {} from group id: {}, topic: {}, partition: {}, offset: {}, thread name: {}",
                payload, groupId, topic, partition, offset, Thread.currentThread().getName());

       	// 使用线程池来处理数据，加快消费速度
		signatureHandlerExecutor.submit(() -> {
			// 这里处理业务逻辑
			ack.acknowledge();
		});
    }
}

```

### 总结

至此,我们继承了消费者以及生产者,实际处理过程中，作为消费者我们是需要记录当前消费的消息的，这样一旦出现了问题我们可以拿到原始消息进行对账或者重放进行人工补偿。

如果你想通过连接工具连接kafka查看消费数据进度，我给你推荐Offset Explorer这个工具。