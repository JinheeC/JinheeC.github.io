---
title: "Kafka Message 추적하기 - Zipkin, Brave"
categories: 
- Kafka
excerpt: |
  카프카 producer 와 consumer 에 데코레이터를 추가해 추적하는 방법이다.
  TracingProducer - 레코드당 producer span 을 만들고 헤더를 통해 전파한다.
  TracingConsumer - poll() 이 실행될때 consumer span 을 만들고 헤더에 span 값이 있으면 trace 를 만들어 완성한다.
feature_text: |
  Zipkin, Brave 를 이용해서 Kafka Client Message 를 추적하는 방법에 대해
---

* table of contents
{:toc .toc}


[Brave Kafka instrumentation](https://github.com/openzipkin/brave/tree/master/instrumentation/kafka-clients)
카프카 producer 와 consumer 에 데코레이터를 추가해 추적하는 방법이다.
* TracingProducer - 레코드당 producer span 을 만들고 헤더를 통해 전파한다.
* TracingConsumer - poll() 이 실행될때 consumer span 을 만들고 헤더에 span 값이 있으면 trace 를 만들어 완성한다.

## 사용법
우선 generic kafka component 를 만든다.
``` java
kafkaTracing = KafkaTracing.newBuilder(tracing)
                           .remoteServiceName("my-service")
                           .build();
```
producer 를 wrapping 해서 사용하려면 아래와 같이 한다.
``` java
Producer<K, V> stringProducer = new KafkaProducer<>(settings);
TracingProducer<K, V> tracingProducer = kafkaTracing.producer(producer);
tracingProducer.send(new ProducerRecord<K, V>("my-topic", key, value));
```

consumer 를 wrapping 해서 사용하려면 아래와 같이 한다.
``` java
Consumer<K, V> consumer = new KafkaConsumer<>(settings);
TracingConsumer<K, V> tracingConsumer = kafkaTracing.consumer(consumer);
tracingConsumer.poll(10);
```

## 내부 동작 방식
일반적으로 3개의 span 으로 메세지를 추적한다.
* producer 가 추적되면 레코드당 PRODUCER span 이 잡힌다. 
* consumer 가 추적되면 poll 할때 받은 레코드나 토픽에 기반한 CONSUMER span 이 만들어진다.
* Message Processor 가 `KafkaTracing.nextSpan` 을 해서 새로운 span 을 나타내는데 사용한다.
producer span, consumer span 은 위에서 언급한 방법으로 TracingProducer, TracingConsumer 를 사용할 경우 brave instrumentation 이 span 을 생성해서 헤더에 넣어준다. 그리고 마지막 span 인 message processor 의 span 은 선택적 이다. consume 후 특정 logic 을 탔는지 확인하려면 nextSpan 으로 span 을 생성해주면 된다.

http와 유사하게 헤더를 사용하여 추적 컨텍스트가 전파된다. http와 달리 메시지 처리 작업(위의 span 종류에서 3번째 span)은 메시지 consuming(2번째 span)과 분리된다. Consumer.poll 은 http 요청과 같은 하나의 메시지를받는 대신 잠재적으로 여러 토픽에서 대량으로 동시에 메시지를 수신하기 때문이다. (기본 max.poll.records 는 500개..) 이러한 이유 때문에 들어오는 tracing context가없는 경우 폴링 호출마다 단일 CONSUMER span이 공유되고 이 span은 KafkaTracing.nextSpan이 호출 될 때 parent 가 된다.

## 메세지가 특정 로직을 타는지 tracing 하는 방법
trace 를 계속 이어나가기 위해 `KafkaTracing.nextSpan` 를 하려면
``` java
// Typically, poll is in a loop. the consumer is wrapped
while (running) {
  ConsumerRecords<K, V> records = tracingConsumer.poll(1000);
  // either automatically or manually wrap your real process() method to use kafkaTracing.nextSpan()
  records.forEach(record -> process(record));
  consumer.commitSync();
}
```

아래와 같이 nextSpan, finish 로 span 을 만들어준다
``` java
<K, V> void process(ConsumerRecords<K, V> record) {
  // Grab any span from the record. The topic and key are automatically tagged
  Span span = kafkaTracing.nextSpan(record).name("process").start();

  // Below is the same setup as any synchronous tracing
  try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) { // so logging can see trace ID
    return doProcess(record); // do the actual work
  } catch (RuntimeException | Error e) {
    span.error(e); // make sure any error gets into the span before it is finished
    throw e;
  } finally {
    span.finish(); // ensure the span representing this processing completes.
  }
}

```

## 예제 
예제 코드는 [깃헙 링크](https://github.com/JinheeC/brave-instrumentation-kafka-example)를 통해 확인할수 있다.

## 결과
produce -> consume.poll 만 했을 경우
![Alt text](https://monosnap.com/image/QspFxgRV3w2QZaC9Hbo047NqTflEfP)

produces -> consume.poll -> consumer logic(doSomething)
![Alt text](https://monosnap.com/image/wDgvDGj62Bdj95Xg6J60Q9nsSJChhU)

produces -> consume.poll(1,2 컨슈머 2개) -> consumer logic(doSomething)(1,2)
![Alt text](https://monosnap.com/image/dU70dKJehFeZUWuAgrperWHBfrmIvy)
