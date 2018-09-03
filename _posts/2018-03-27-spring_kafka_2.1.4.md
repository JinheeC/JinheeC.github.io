---
title: "Spring Kafka 2.1.4"
categories: 
- Kafka
excerpt: |
  Spring kafka 에서 토픽만들기, 메세지 보내기(produce) 메세지 받기 (consume) 등 가장 기본이 되는 내용에 대한 정리입니다.
feature_text: |
   Spring Kafka 2.1.4 의 기본 내용에 대해
feature_image: "https://picsum.photos/2560/600?random"
---

* table of contents
{:toc .toc}

****
## 토픽 만들기

`kafkaAdmin`을 Bean으로 등록해 놓으면 Bean으로 등록 된 `Topic`을 브로커에 추가 해 준다.
``` java
    @Bean
    public KafkaAdmin admin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_ADDR);
        return new KafkaAdmin(configs);
    }

    @Bean
    public NewTopic topic1() {
        return new NewTopic("test1", 10, (short) 1); 
    }

    @Bean
    public NewTopic topic2() {
        return new NewTopic("test2", 10, (short) 1);
    }
```
* Topic(토픽 이름, 파티션 개수, replication Factor)
* KafkaAdmin이 초기화 될 때 이 컨피그 값을 가지고 그에 맞도록 AdminClient 를 생성 하고 토픽을 만들도록 함.

> NOTE: ***KafkaAdmin과 AdminClient의 차이***
>
> KafkaAdmin:  AdminClient 한테 토픽을 만들라고 위임하는 어드민.
>
> AdminClient:  토픽, 브로커, 등 설정을 전반적으로 관리 함.
> ```
> INFO Topic creation {"version":1,"partitions":{"8":[0],"4":[0],"9":[0],"5":[0],"6":[0],"1":[0],"0":[0],"2":[0],"7":[0],"3":[0]}} (kafka.admin.AdminUtils$)
> INFO Topic creation {"version":1,"partitions":{"8":[0],"4":[0],"9":[0],"5":[0],"6":[0],"1":[0],"0":[0],"2":[0],"7":[0],"3":[0]}} (kafka.admin.AdminUtils$)
> ```

****

## 메시지 보내기 (Produce)

#### KafkaTemplate 사용
`KafkaTemplate`은 `Producer`를 한번 감싼 클래스이고 카프카 토픽으로 데이터를 보낼 수 있는 편리한 메소드들을 제공한다. 
![Alt text](https://monosnap.com/file/0s1OJ2bvsznfIIx4KEki4ZtA2VJYYV.png)


***카프카 탬플릿을 사용하려면 카프카 탬플릿의 생성자 안에서 Producer Factory가 생성 할 Producer를 설정해서 생성하면 된다.***
``` java
@Configuration
public class ProducerConf {
    private static final String BOOTSTRAP_ADDR = "localhost:9092";

    @Bean
    public KafkaTemplate<Integer, String> kafkaTemplate() {
        return new KafkaTemplate<Integer, String>(new DefaultKafkaProducerFactory<>(producerConfigs()));
    }

    @Bean
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_ADDR);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return props;
    }
}
```

위 코드로 카프카 탬플릿이 설정되었고 사용하려면 단순히 아래에 나열된 카프카 탬플릿의 메소드를 아무거나 호출하면 된다. 

``` java
send(String topic, V data)
send(String topic, K key, V data)
send(String topic, Integer partition, K key, V data)
send(String topic, Integer partition, Long timestamp, K key, V data)
send(ProducerRecord<K, V> record)
send(Message<?> message)
```
리턴값은 ListenableFuture<SendResult<key, data>> 이고 Future를 상속한 클래스이다. 콜백이 설정되어있을때 future가 끝나면 그 콜백이 즉시 실행된다.

> NOTE: ***ProducerRecord와 Message의 차이***
>
> ProducerRecord: 카프카로 보내지는 레코드로써 `토픽, 파티션, key, value, 헤더, 타임스탬프`를 가진다.
>
> Message: Spring의 Message 객체로써 헤더와 페이로드를 가지고 있다. 이 경우 message의 헤더 부분에 `토픽, 파티션, key, 타임스탬프`를 가지고 있게 된다. `value(data)`는 payload 에 들고 있다. 즉 ProducerRecord의 값을 헤더와 페이로드로 나눠서 가지고 있게 됨.
>
>
> NOTE: ***send시 비동기 콜백 사용하기*** 
>
> ListenableFuture에 ListenableFutureCallback 을 등록해서 하나의 send 에 대해서 전송이 완료되었을 때 콜백을 설정할 수 도 있고 KafkaTemplate에 ProducerListener를 등록해서 KafkaTemplate의 모든 send 에 대해서 전송이 완료되었을 때 콜백을 설정할수 있다. 
>
> ``` java
> void onSuccess(String topic, Integer partition, K key, V value, RecordMetadata recordMetadata);
>
> void onError(String topic, Integer partition, K key, V value, Exception exception);
>
> boolean isInterestedInSuccess();
> ```
> 
> NOTE: 디폴트 ***Partitioning 전략***
>
> * 파티션이 정해졌다면 정해진 파티션으로 파티셔닝
> * 파티션이 정해지지 않았을때
>    * Key가 정해졌다면 key 의 hash 값을 기준으로 파티셔닝
>    * key도 정해지지 않았다면 Round-Robin 방식으로 파티셔닝
>     
> NOTE: KafkaTemplate의 send() 마다 하는 일.
>
> `트랜젝션이 아니면` Producer를 생성해서 Producer 가 record를 send 하도록 시킨다. 그리고 send 가 끝나고 돌아오는 RecordMetadata 로 SendResult 를 만들고 만약에 ProducerListener 로 설정해 줬던 콜백이 있다면 해당 콜백을 실행시켜준다. 트랜잭션이 아닐 경우 Producer는 1개다.
>  
> NOTE:  설정에 따른 timestamp 값
> 
> 토픽이 `CREATE_TIME` 을 사용하도록 설정이 되어있으면 
>
>	* 타임스탬프는 파라미터로 넘겨주는 값을 사용하거나
>
>	* 타임스탬프를 넘겨주지 않았다면 생성시킨다. 
>
>
> 토픽이 `LOG_APPEND_TIME` 을 사용하도록 설정되어있으면
>
>	* 파라미터로 주어진 타임스탬프는 무시되어지고 브로커가 브로커 로컬타임을 사용해서 메세지를 만들게 된다.

그 이외의 메소드
``` java
Map<MetricName, ? extends Metric> metrics();

List<PartitionInfo> partitionsFor(String topic);
```
위 두 메소드는 단순히 Producer 한테 해당 동작을 위임한다.

> NOTE: ***sendResult***에는 ProducerRecord와 RecordMetadata(ack) 가 있다. 
>
> NOTE: send 의 결과로 리턴되는 SendResult 값을 바로 받으려면 쓰레드를 값이 도착할 때 까지 멈추고 기다리면되는데 기본적으로 future가 가지고 있는 get()을 활용하거나 카프카 탬플릿의 flush()를 사용할 수도 있다. auto flush 기능을 사용하면 성능 저하가 있으므로 주의.


#### ReplyingKafkaTemplate 사용

***카프카 탬플릿을 상속받은 ReplyingKafkaTemplate을 사용하면 요청/응답 구조를 사용 할 수 있다.***
``` java
RequestReplyFuture<K, V, R> sendAndReceive(ProducerRecord<K, V> record);
```
ReplyingKafkaTemplate 클래스에는 하나의 메소드가 있는데 KafkaTemplate의 send()와의 차이점은 단순히 토픽에 메세지를 Produce 하는 것 뿐 아니라 컨슈머가 해당 메시지를 받았을 때 응답 메시지를 받을 수 있다는 것이다. 응답 메시지를 받기 위해서는 카프카 produce와 마찬가지로 (리플라이 용)토픽이 필요하고 그 설정은 아래와 같이 가능하다.
``` java
//test1 : produce 할 topic
//test2 : reply 할 topic
ProducerRecord<String, String> record = new ProducerRecord<>("test1", "foo");
record.headers().add(new RecordHeader(KafkaHeaders.REPLY_TOPIC, "test2".getBytes()));
```
Produce 할 topic이 설정된 레코드의 헤더에 응답메시지가 들어갈 토픽 이름을 넣어주고 sendAndReceive 하면 되는데, 이때 ReplyingKafkaTemplate이 헤더에  `CORRELATION_ID` 를 같이 넣어준다. CORRELATION_ID 을 보고 Consumer 쪽에서 요청 메시지와 응답메시지를 매칭 하는데 사용되어진다. 해당 토픽을 구독한 Consumer 쪽 코드를 보면 아래와 같다.
``` java
@KafkaListener(topics = "test1")
@SendTo
public String listen(String in) {
    System.out.println("Server received: " + in);
    return in.toUpperCase();
}
```

> NOTE: ***RequestReplyFuture***(from sendAndReceive)와 ***ListenableFuture***(from send)의 ***차이***
>
> * RequestReplyFuture: RequestReplyFuture 로 SendResult 와 ConsumerRecord를 얻을 수 있다.
>
> * ListenableFuture: ListenableFuture로 SendResult를 얻을 수 있다.
>
> NOTE: ***@SendTo*** 사용법
>
> @SendTo : 리턴값이 헤더에 들어있는 Reply 용 Topic으로 Reply(Produce) 된다.
>
> @SendTo("topicA") : 리턴값이 topicA 라는 토픽으로 Reply(Produce) 된다.

****
## 메시지 받기 (Consume)

메시지를 받으려면 MessageListenerContainer를 설정해 주고 메시지 Listener를 만들어 주면 된다. 간단하게 메소드에 `@KafkaListener` 어노테이션을 붙이면 리스너가 된다. 
MessageListenerContainer의 구현체는 2가지가 있는데 `KafkaMessageListenerContainer`와 `ConcurrentMessageListenerContainer` 가 있다. 
* KafkaMessageListenerContainer의 경우 하나의 쓰레드에서 동작하고 모든 메시지들을 이 하나의 컨테이너에서 처리한다. 
* ConcurrentMessageListenerContainer 의 경우 여러개의 KafkaMessageListenerContainer를 만들어서 다중 쓰레드로 메시지를 처리할 수 있도록 위임하게 된다. Concurrency가 3이면 KafkaMessageListenerContainer도 3개 생성되고 각 컨테이너에 파티션이 균등 할당된다. 

#### ListenerContainer 사용

``` java
@EnableKafka
@Configuration
public class ConsumerConf {
    private static final String BOOTSTRAP_ADDR = "localhost:9092";

    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<Integer, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.setMessageConverter(new StringJsonMessageConverter());
        factory.getContainerProperties().setPollTimeout(3000);
        return factory;
    }

    @Bean
    public ConsumerFactory<Integer, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_ADDR);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(props);
    }

}
```
메시지는 리스너가 poll() 할 때마다 리스너에게 전달되는데 메시지 처리가 끝나면 commit 을 통해 현재 offset을 바꿔주게 된다. commit 이 이루어지는 시점은 설정에 따라 달라지는데 `enable.auto.commit` 이 true 로 설정되어있다면 설정된 interval 마다 자동으로 커밋하게 된다. (기본은 true)

> NOTE: 정확한 ***commit 시점***을 정하려면
>
> enable.auto.commit 를 false 로 설정하고 아래의 AckMode 중 하나를 선택하고 설정면 된다.
> 
> RECORD = 리스너가 레코드를 프로세싱 하고 나서 오프셋 커밋
>
> BATCH = poll 이 다 처리가되고 모든 레코드가 리턴되면 오프셋 커밋
>
> TIME = (마지막 커밋 이후의 시간이 ackTime보다 작으면)poll 이 다 처리되고 모든 레코드가 리턴될 때 오프셋 커밋
>
> COUNT = (마지막 커밋 이후 레코드 개수가 받아지는 한)  poll 이 다 처리되고 모든 레코드가 리턴 되었다면 오프셋 커밋
>
> COUNT_TIME = 두개 조건 만족
>
> MANUAL = ack 도착 하면 BATCH 처럼 행동함.
>
> MANUAL_IMMEDIATE =  리스너가 Acknowledgment.acknowledge() 부를때 바로 커밋
>
> NOTE: max.poll.records 프로퍼티로 한번에 가져오는 poll 개수 조정 가능

`@KafkaListener` 를 메소드에 붙여서 리스너를 만들고 토픽을 Consume 하는 코드는 아래와 같다. 이 어노테이션을 사용하려면 @Configuration 클래스 중 하나에 `@EnableKafka` 어노테이션을 달아 줘야 한다. 이 어노테이션은 KafkaListenerContainerFactory를 찾기 때문에 보통 ListenerContainer 설정하는 클래스에 같이 달아준다.
``` java
@KafkaListener(topics = "test1")
public void listen(String data) {
    System.out.println(data + " is consumed.");
}
```
컨테이너가 리스너에게 Spring 의 Message 타입으로 넣어주기 때문에 파라미터에는 메시지로 보냈던 Value 부분 즉, payload 에 해당하는 데이터 타입으로 받는다. 만약 Value 가 아닌 다른 정보가 필요하다면 헤더에서 값을 찾으면 된다. 그 과정 대신에 어차피 Message로 보내주기 때문에 Message로 받아도 가능하다.

> NOTE: 특정 파티션 설정 및 초기 오프셋 설정 가능. 초기 오프셋을 설정해 주면 어플리케이션이 새로 시작할 때 마다 초기 오프셋 부터 읽게 된다.
>``` java
> @KafkaListener(id = "bar", topicPartitions =
>   { @TopicPartition(topic = "topic1", partitions = { "0", "1" }),
>     @TopicPartition(topic = "topic2", partitions = "0",
>     partitionOffsets = @PartitionOffset(partition = "1", initialOffset = "100"))
>   })
> ```
> 
> NOTE: ***헤더*** 값을 얻으려면 아래와 같이 얻을 수 있다. 
>``` java
> public void listen(@Payload String value,
>       @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) Integer key,
>       @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
>       @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
>       @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long ts
>       )
>```
> 
> NOTE: 배치로 ***한번에 모든 레코드 받기*** 
>
> 기본으로는 배치가 아니기 때문에 한번 Poll()해서 가져온 Record 들이 하나씩 리스너 메소드로 들어오게 되는데 아래와같이 설정해 주면 리스트 타입으로 한번에 받을 수 있다. (header도 리스트로 들어온다.)
> ``` java
> @Bean
> public KafkaListenerContainerFactory<?> batchFactory() {
>    ConcurrentKafkaListenerContainerFactory<Integer, String> factory =
>            new ConcurrentKafkaListenerContainerFactory<>();
>    factory.setConsumerFactory(consumerFactory());
>    factory.setBatchListener(true);  // <<<<<<<<<<<<<<<<<<<<<<<<< 배치 true
>    return factory;
> }
> // 리스너 부분
> @KafkaListener(topic="test1")
> public void listen(List<String> list) {
>    ...
> }
> ```

`@KafkaListener` 가 클래스레벨에 있으면 메소드레벨에 `@KafkaHandler`를 달아야 한다. 메시지가 전달되어지면 변경된 메시지 페이로드 타입이 어떤 메소드로 콜할지 판단해서 들어간다. 즉 메시지의 데이터 타입에 따라 메소드로 분리 처리가 가능하다. ***isDefault*** 값을 설정해 놓으면 어떤 메소드 파라미터도 맞지 않을때 해당 메소드로 호출된다.
``` java
@KafkaListener(topics = "test1")
static class MultiListenerBean {

    @KafkaHandler
    public void listen(String foo) {
        ...
    }

    @KafkaHandler
    public void listen(Integer bar) {
        ...
    }

    @KafkaHandler(isDefault = true)
    public void listenDefault(Object object) {
        ...
    }

}
```
이미 이 클래스에서 메소드를 찾을때 부터 도메인에있는 오브젝트 타입으로 페이로드 부분이 전환되어있어야 한다. 그러기 위해서 deserializer를 따로 만들어서 사용하거나 카프카의 StringJsonMessageConverter를 사용하면된다. 

#### StringJsonMessageConverter 사용
기본으로 제공되어지는 ***StringJsonMessageConverter***는 TypePrecedence 가 있어서 @KafkaListener가 붙어있는 메소드의 파라미터 타입이 메세지 컨버터에게 가서 그 타입에 맞게 deserialize 된다. 
카프카 탬플릿에 바로 주입해서 사용할 수 있다. 

> NOTE: StringJsonMessageConverter 의 ***타입 추론***
>
> 위에 언급된 것처럼 메소드의 파라미터 타입을 보고 추론하는 것이기 때문에 @KafkaListener가 클래스 레벨에 붙어 있고 메소드에는 @KafkaHandler를 쓰는 상황에서는 타입추론이 불가능하다. 그래서 이 경우에는 TypePrecedence 를 TYPE_ID 로 사용해야 한다. (default 는 TypePrecedence.INFERRED) TypePrecedence 타입이 INFERRED 가 아니면 헤더에서 데이터타입을 찾도록 로직이 구현되어있다. 즉 ***메소드레벨, 클래스 레벨 또는 인터페이스 레벨에 @KafkaListener 를 사용***할 수 있고 대신 메소드레벨이 아니라면 헤더에 변환해야 할 데이터 타입을 명시해줘야 한다.


#### Container 의 lifecycle
@KafkaListener로 만든 리스너는 Bean이 아니다. 대신에 이 리스너들은 `KafkaListenerEndpointRegistry` 라는 Bean에 등록이 되어서 관리되고 있는데 이 Bean이 ***컨테이너 들의 라이프 사이클을 제어***할 수 있다. autoStartUp true 로 되어있는 컨테이너들은 KafkaListenerEndpointRegistry가 자동으로 시작시킬 수도 있고 멈추게도 할 수 있다. 이 과정은 코드로 가능하고 모든 팩토리에서 생성된 모든 컨테이너들이 같은 패이스로 제어된다. 만약 특별히 다른 패이스로 제어하고 싶다면 @KafkaListener 어노테이션 속성으로 아래와 같이 autoStartUp 을 오버라이딩 하고 id를 지정해 준다. 이렇게 리스너를 설정해 주면 이후에 KafkaListenerEndpointRegistry에서 이 컨테이너를 찾아서 start, stop이 가능하다.

``` java
@Autowired
private KafkaListenerEndpointRegistry registry;

@KafkaListener(id = "myContainer", topics = "test1", autoStartup = "false")
public void listen(...) { ... }

//설정 된 id로 컨테이너를 찾아 시작시킨다. 
registry.getListenerContainer("myContainer").start();
```
****
## 메시지 재시도하기  (Retry)

리스너가 예외를 던지면 처리할 수 있는 두가지 옵션이 존재한다. 첫 번째로 재시도가 있고, 두 번째로 에러 핸들러에 넘기는 방법이 있다. 두 방법 다 설정을 해줘야 사용할 수 있다. 재전송 하려면 RetryingMessageListenerAdapter 를 사용하면 된다. `RetryTemplate`  와 `RecoveryCallback<Void>`를 설정해서 사용할 수 있다.
``` java
    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<Integer, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        ...
        factory.setRetryTemplate(new RetryTemplate());
        factory.setRecoveryCallback(new MyRecoveryCallback());
        
        ...
        return factory;
    }
```
* 정상적인 동작은 설정된 횟수 만큼 재시도를 하다가 최종실패 시 recovery 로직을 타고 그 로직에서도 예외가 났다면 그 예외는 컨테이너 레벨로 전달되어진다.
* 만약 RecoveryCallback 이 주어지지 않았다면 재시도를 계속 하다가 리스너의 예외가 컨테이너한테 던져진다. 
* 그러면 그때 (에러핸들러가 설정되어있을 경우) 에러핸들러가 호출된다. 

@KafkaListener를 사용하고 RetryTemplate와 recoveryCallback을  Container Factory에 설정하면 리스너는 적절하게 재시도를 할 것이다.
``` java
public class SimpleRetryPolicy implements RetryPolicy {

	/***
	 * The default limit to the number of attempts for a new policy.
	 */
	public final static int DEFAULT_MAX_ATTEMPTS = 3;
```
retryPolicy를 생성해서 원하는 대로 설정 할 수 있는데 RetryTemplate이 생성 될 때 default retry 정책은 SimpleRetryPolicy 이다. 기본으로 설정된 시도 횟수는 3회이기 때문에 listener는 3회 동안 메시지 프로세싱을 실패하면 RecoveryCallback 메소드로 보낸다.
> NOTE: RetryContext가 처리못한 레코드를 들고다니는데 이 컨텍스트의 내용은 리스너의 타입에 달렸다. 만약에 리스너가 acknowledging 이나 consumer aware 하다면 추가적인 속성으로 Acknowledment, Consumer를 들고 다닐 것이다. 

## 에러 처리하기

retry 와 마찬가지로 에러핸들러는 기본으로 설정되어있지 않다. 리스너 레벨의 Error Handler가 있고 컨테이너 레벨의 Error Handler 가 있다. 
* @KafkaListener 속성으로 errorHandler 가 있어서 이걸 설정해 주면 리스너 레벨에서 사용할 수 있다.
* 컨테이너 팩토리에 errorHandler 를 설정해 주면 컨테이너 레벨에서 사용가능하다. 
``` java
1) 리스너 레벨
@KafkaListener(topics = "annotated23", errorHandler = "replyErrorHandler")
2) 컨테이너 레벨
concurrentKafkaListenerContainerFactory.getContainerProperties().setErrorHandler(myErrorHandler);
```

리스너 어노테이션이 붙은 메소드가 처리하지 못하고 예외를 던지면 컨테이너로 다시 던져질 것이고 메세지는 컨테이너 설정에 해준것 대로 에러핸들링이 될 것이다.
> NOTE: error 발생 시 offset 조정
>
> 컨테이너 레벨에 에러핸들러든 리스너 레벨의 에러핸들러ConsumerAwareListenerErrorHandler ConsumerAwareContainerErrorHandler가 있는데 이 구현체로 에러 발생시 offset 조정을 할 수 있다. (대신 컨테이너 레벨의 offset 조정시에는 ackOnError를 false로 설정해야 한다.)

## 예제 소스
[위의 내용이 들어있는 예제소스](https://github.com/JinheeC/springKafkaTutorial) 에서 자세한 코드를 확인할 수 있다.
