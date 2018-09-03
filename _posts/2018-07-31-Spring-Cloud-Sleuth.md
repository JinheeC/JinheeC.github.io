---
title: "Spring Cloud Sleuth 로 Spring Kafka 메세지 Tracing 하기"
categories: 
- Kafka
feature_text: |
  Spring Cloud Sleuth 로 Zipkin tracing 을 하는 방법에 대해
---

# Spring Cloud Sleuth

* table of contents
{:toc .toc}

카프카에 zipkin 을 연동하기 위해서 사용하게 되었다. 

## Brave Instrumentation for Kafka Client 의 불편함
기존의 Brave-Instrumentation for kafka client 의 경우 spring kafka 와 같이 쓰려면 spring kafka 가 한번 wrapping 하게 하기위해서 DefaultKafkaProducerFactory, DefaultKafkaConsumerFactory 를 상속 받아서 아래와 같이 직접 wrapping 을 해야 했다.

``` java
public class CustomKafkaConsumerFactory extends DefaultKafkaConsumerFactory {

    public EdaKafkaConsumerFactory(Map configs) {
        super(configs);
    }

    @Override
    public Consumer createConsumer(String groupId, String clientIdPrefix, String clientIdSuffix) {
        return KafkaTracer.consumer(super.createConsumer(groupId, clientIdPrefix, clientIdSuffix));
    }
}

```

``` java
public class CustomKafkaProducerFactory extends DefaultKafkaProducerFactory {

    public EdaKafkaProducerFactory(Map configs) {
        super(configs);
    }

    @Override
    public Producer createProducer() {
        return KafkaTracer.producer(super.createProducer());
    }
}

```

참고로 위 코드에서 사용하는 KafkaTracer 는 아래와 같다.
``` java
public class KafkaTracer {
    private static final KafkaTracing kafkaTracing = KafkaTracing.create(Tracing.newBuilder()
                                                                                .localServiceName("EDA-GATEWAY")
                                                                                .currentTraceContext(new StrictCurrentTraceContext())
                                                                                .spanReporter(AsyncReporter.create(URLConnectionSender.create("http://zipkin-url:9411/api/v2/spans")))
                                                                                .build());

    public static Consumer<?, ?> consumer(Consumer<?,?> consumer) {
        return kafkaTracing.consumer(consumer);
    }

    public static Producer<?, ?> producer(Producer<?, ?> producer) {
        return kafkaTracing.producer(producer);
    }

    public static Span nextSpan(ConsumerRecord<?, ?> record) {
        return kafkaTracing.nextSpan(record);
    }
}
```

그런데 Spring Cloud Sleuth 의 경우 위의 구현이 이미 되어있었다.  [KafkaTracing 을 생성하는 블록](https://github.com/spring-cloud/spring-cloud-sleuth/blob/9a085e3b415bf4f18ae34c9cd72ccf9173bd12f5/spring-cloud-sleuth-core/src/main/java/org/springframework/cloud/sleuth/instrument/messaging/TraceMessagingAutoConfiguration.java#L101) 과 [Spring AOP 로 Consumer, Producer를 wrapping 하는 부분](https://github.com/spring-cloud/spring-cloud-sleuth/blob/9a085e3b415bf4f18ae34c9cd72ccf9173bd12f5/spring-cloud-sleuth-core/src/main/java/org/springframework/cloud/sleuth/instrument/messaging/TraceMessagingAutoConfiguration.java#L151) 의 코드를 보면 알 수 있다.

## 코드에 적용하기

maven 또는 gradle 디팬던시로 `compile ('org.springframework.cloud:spring-cloud-starter-zipkin')` 를 추가하고 아래와 같이 yml 설정파일을 설정해주면 굉장히 간단하게 zipkin 트래이싱을 zipkin 서버에서 확인할 수 있다. 

``` java
spring:
  zipkin:
    base-url: http://zipkin-server.com:9411 
  sleuth:
    sampler:
      probability: 1
```
probability 는 실제 통신하는 메세지(또는 요청) 중 몇개를 보여주느냐를 말하는데 1이면 100% 0.3 이면 30% 이다. 테스트를 해볼때는 1로 해두고 확인하는 것이 편하다.

### zipkin 서버
빠른 설치가 필요하다면 아래로 설치

``` 
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```
