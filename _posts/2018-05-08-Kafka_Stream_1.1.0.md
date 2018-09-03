---
title: "Kafka Streams 1.1.0"
categories: 
- Kafka
excerpt: |
  이벤트 시간 `Event time`: 이벤트가 발생했거나 데이터 레코드가 발생한 시점. 즉, 해당 이벤트를 만들어서 카프카에 주는 그 발생 주체에서 이벤트가 발생된 시점을 말한다. 
  처리시간 `Processing time`: 데이터 레코드가 처리되는 시점, 레코드가 소비되는 시점. 처리시간은 원래 이벤트 시간보다 몇 ms, 몇 s, 또는 몇 hour 이후일 수 있다. 
  등록시간 `Ingestion time`: 데이터 레코드가 카프카 브로커에 의해 토픽 파티션에 저장되는 시간. 
feature_text: |
  Kafka Streams 1.1.0 공식문서 내용입니다.
---
# Kafka stream
공식문서 번역 + 이해한 내용 첨언한 글입니다.

* table of contents
{:toc .toc}


카프카 스트림은 데이터를 처리하고 분석하기 위한 클라이언트 라이브러리다. 

## CORE CONCEPT
### *TIME* Stream 의 시간 개념
* 이벤트 시간 `Event time`: 이벤트가 발생했거나 데이터 레코드가 발생한 시점. 즉, 해당 이벤트를 만들어서 카프카에 주는 그 발생 주체에서 이벤트가 발생된 시점을 말한다. 
* 처리시간 `Processing time`: 데이터 레코드가 처리되는 시점, 레코드가 소비되는 시점. 처리시간은 원래 이벤트 시간보다 몇 ms, 몇 s, 또는 몇 hour 이후일 수 있다. 
* 등록시간 `Ingestion time`: 데이터 레코드가 카프카 브로커에 의해 토픽 파티션에 저장되는 시간. 

이벤트 시간과 등록 시간의 선택은 실제로 카프카(스트림아니고)의 컨피그를 통해 이루어진다. 타임스탬프가 자동으로 붙게되는데 카프카가 어떻게 컨피그되어졌는가에 따라 타임스탬프카 이벤트 시간을 나타낼 것인지 등록시간을 나타낼 것인지를 선택할 수 있다. 

`TimestampExtractor` 라는 인터페이스가 새로운 레코드가 들어올때 이벤트에 맞는 시간을 넣어주게 되는데 이걸 **data-driven-time** 또는 **stream time** 이라고 부른다. 그리고 카프카에는 실제 시간도 있기 때문에 예를 들어 이벤트 시간을 가지고 있는 레코드를 이벤트 시간을 기반으로 타임스탬프를 검색하거나 계산하고 현재 실제 시간을 반환함으로써 원하는 프로세싱 원하는 비즈니스 로직을 넣을 수 있다.

*타임스탬프값(event-time) 은 상황에 따라 다르다.*
* 인풋 레코드를 처리해서 새로운 아웃풋 레코드가 생기면 process() 메소드안에서 context.forward()가 실행되어지는데 아웃풋 레코드의 타임스탬프 값은 인풋 레코드의 타임스탬프 값을 그대로 물려받게된다. (해당 이벤트가 시작된 시간이니까 같아야 하는게 당연.)
* Punctuator.punctuate()와 같은 주기적인 메소드로 부터 생성된 새로운 아웃풋 레코드의경우 타임스탬프는 현재 내부 시간. (context.timestamp()로 부터 얻어지는 시간)이 된다. (이때 생겼으므로 어떻게 보면 당연한 일.)
* aggregate되어진 업데이트된 레코드 결과로서 타임스탬프는 업데이트 하도록 트리거한 가장 마지막에 도착된 인풋 레코드의 값이 될 것이다. 

### *STATES* Stream 의 상태 개념
몇몇의 스트림 어플리케이션은 상태가 필요하지 않을 수 있다. 즉, 다른 메세지 처리와 무관하게 돌아갈 수도 있다. 하지만 상태를 유지하고 인지하는 것은 정교한 스트림 처리가 가능하도록 한다. 이를 위해 여러 상태 저장 연산자가 `카프카 스트림 DSL` 로 제공이 되고있다.
카프카는 **State Stores** 를 제공하고 있다. 이것은 데이터를 저장하고 쿼리하기위해 사용되어지고 *상태 저장작업을 구현*할 때 중요하다. 카프카 스트림의 모든 Task는 처리에 필요한 데이터를 저장하고 쿼리하기위해서 하나 또는 그 이상의 state store 를 가지고 있는데 이 스테이트 스토어는 api를 통해 액세스 할 수 있다. State Store 는 영구적인 키 벨류 저장소 또는 메모리 내 해시 맵, 아니면 다른 편리한 데이터 구조가 될 수 도 있다. 카프카 스트림은 `Fault Tolerance` 하고 자동 복구되는 `Local State Store`를 제공한다. 

카프카 스트림을 사용하면 스테이트 스토어를 읽을 수 있다. (READ-ONLY: 메소드, 쓰레드, 프로세스, 또는 스테이트 스토어를 만든 어플리케이션 외부에 있는 다른 어플리케이션 으로도 Read-Only 가능하다.)

### *Processing Guarantees*
카프카 스트림은 정확히 한번 데이터를 전달하는 것을 보장한다. 

## ARCHITECTURE

카프카 스트림은 어플리케이션 개발을 간단하게 되도록 했다. 카프카 프로듀서와 컨슈머 라이브러리를 만들고 데이터 병렬, 분산 코디네이션, 폴트 톨러런스, 운영 단순성을 제공하는 카프카의 고유 능력을 사용하도록 했다. 
그렇다면 카프카는 내부에서 어떻게 동작하고 있을까?
![Alt text](https://monosnap.com/image/QkVHkknjidbwqXtjqJzsMpVzdjUKiM)

### *Stream Partitions and Tasks* 파티션과 테스크
카프카의 메세징 계층은 데이터 저장 및 운반을 위해서 파티션을 나누게되고 메세지를 처리하는 어플리케이션 쪽에서는 데이터를 처리하기위해서 파티션을 스트리밍한다. (파티셔닝을 하는 이유는 데이터 인접성, 탄력성, 확장성, 높은 성능 및 내결함성을 가능하게 하기위해서다.) 
카프카 스트림은 `Partition` 과 `Task` 라는 개념을 사용하는데 **Task 는 카프카 토픽 파티션에 기반한 병렬 모델의 논리적 단위**를 뜻한다. 
* 각각의 스트림 파티션은 카프카 토픽 파티션에 매핑되고 데이터 레코드의 순서로 정렬되어져 있다. 
* 스트림의 데이터 레코드는 해당 토픽의 카프카 메세지로 매핑된다. 
* 데이터 레코드의 키는 카프카와 카프카 스트림에서 데이터의 파티셔닝을 결정한다. 

하나의 어플리케이션의 프로세서 토폴로지는 이를 여러개의 Task 로 분할하여 스케일을 조정한다. 좀더 구체적으로 말하자면, 카프카 스트림은 인풋 스트림의 파티션개수에 기반한 고정된 수의 Task 를 만드는데 각각의 Task 는 인풋 스트림의 파티션 리스트를 할당한다. 테스크는 자체 프로세서 토폴로지를 인스턴스화 할수 있다. 그리고 레코드 버퍼에서 각 할당된 파티션 및 프로세스 메세지에 대해 버퍼를 유지한다. 결과적으로 스트림 테스크는 독립적으로 처리되어진다. 
![Alt text](https://monosnap.com/image/Raey0Pwd8ehADwdLhG4Tk6hFreQGb2)

### *THREADING MODEL*
카프카 스트림은 하나의 어플리케이션에서 병렬처리를 사용할 수 있도록 쓰레드 개수를 설정할수 있도록 하고 있다. 각각의 쓰레드는 하나 이상의 Task 들을 실행할 수 있다. 아래 그림은 하나의 스트림 쓰레드가 두개의 스트림 Task 를 돌리고 있는 그림이다. 
![Alt text](https://monosnap.com/image/SvkaYKqkPgjyve8V0koVp2ejWaKqFU)

더 많은 스트림 스레드를 시작하는 것이든 더 많은 어플리케이션 인스턴스를 만드는 것이든 간에 토폴로지를 복제해서 카프카 파티션의 각기 다른 부분을 처리하도록 병렬화 하는 것에 불과하다. 쓰레드 간에 공유된 상태가 없으니까 조정이 필요하지 않고 이 점이 토폴로지를 병렬로 돌리는 것을 간단하게 만든다. 다양한 스트림 쓰레드 사이에서 카프카의 토픽 파티션 할당은 카프카 스트림의 coordination 기능을 사용해서 투명하게 처리된다. 

### *Local State Store*

카프카 스트림은 데이터를 저장하고 쿼리하기 위해 사용될 수 있는 State Store 를 제공한다. 이 개념은 Stateful 한 실행을 할때 중요하다. 자동으로 State Store 를 만들고 관리할 수 있는 카프카 스트림 DSL인 `join()`, `aggregate()`, `windowing`을 사용하면 된다. 카프카 스트림 어플리케이션의 각각의 스트림 Task 는 하나 또는 그 이상의 로컬 스테이트 스토어를 가지는데 API 로 저장하고 쿼리할 수 있다. 카프카 스트림은 `Local State Store`의 내결함성과 자동 복구를 제공한다. 

그럼 계속 얘기가 나오는 내결함성이란 무슨 뜻일까

### *Fault Tolerance* 내결함성
카프카 스트림은 기본적으로 카프카에 통합되어있는 내결함성을 기반으로 한다. 카프카 파티션은 고가용성이고 복제가 될 수 있기 때문에 어플리케이션이 fail 하고 다시 프로세싱이 필요하다해도 카프카 스트림 데이터는 유지되어진다. 카프카 스트림의 Task 는 fail를 핸들링하기 위해서 카프카 컨슈머로부터 제공되어지는 내결함성 능력을 사용한다. 그래서 만약에 **한 기계에서 돌고 있는 Task 가 실패하면 카프카 스트림은 자동으로 같은 어플리케이션이 돌고있는 머신중 하나에서 테스크를 재시작한다.**  

카프카 스트림은 로컬 스테이트 스토어가 실패에 견고하도록 만들었다. 각각의 State Store를 위해서 카프카 스트림은 업데이트를 추적하는 복제된 changelog Kafka topic을 유지한다. 이런 changelog 토픽은 각각의 로컬 스테이트 스토어 인스턴스와 같이 파티션되어진다. compaction 이 적용되어 changelog 가 무한정 커지는 것을 막는다. 시스템 fail 이 나면 다른 인스턴스에서 해당 changelog  를 읽어서 State Store 를 복원하고 해당 Task 를 실행한다. 

위와 같은 Task 재 초기화 비용은 일반적으로 State Store 를 복원하는 시간에 의존한다. 그러므로 이 시간을 최소화 해야 하는데 그러기 위해서는 local State Store 의 복제본을 갖도록 구성할 수 있다. 그래서 카프카 스트림은 이 State Store 를 복구해야 하는 일이 생기면 해당 State Store 의 복제본을 가지고 있는 인스턴스한테 작업을 할당하도록 시도한다. 



## Streams application 작성하기
Kafka Streams API 를 사용하면 프로세서 토폴로지를 만들 수 있다. Streams API 에는 두종류가 있는데 아래와 같다.
* Kafka Streams DSL
	* 고수준 API 이다. 
	* 가장 빈번하게 사용될만한 데이터 변형 기능을 제공한다.
	* map, filter, join, aggregations 가 그에 해당한다.
* Processor API
	* 저수준 API 이다.
	* State Store 와 직접적으로 상호작용할 수 있도록 프로세서를 정의할 수 있도록 한다.
	* DSL 보다 더 유연한 기능을 제공하지만 좀더 개발 코드양이 증가할 수 있다.

``` java
// 단어수 카운트 예제
...
	KafkaStreams streams = new KafkaStreams(createTopology(), config);
	streams.start();
...
public void createTopology(){
	StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> textLines = builder.stream("word-count-input");
    KTable<String, Long> wordCounts = textLines
		                .mapValues(textLine -> textLine.toLowerCase())
		                .flatMapValues(textLine -> Arrays.asList(textLine.split("\\W+")))
		                .selectKey((key, word) -> word)
		                .groupByKey()
		                .count(Materialized.as("Counts"));

    wordCounts.toStream().to("word-count-output", Produced.with(Serdes.String(), Serdes.Long()));
    return builder.build();
}
```

토폴로지를 만들고 난 후 streams.start() 를 해줘야 스트림 쓰레드가 시작된다. 만약 unexpected 예외를 잡아야 한다면, 어플리케이션 시작 전에 아래와 같이 예외 핸들러를 등록해준다. 이 핸들러는 스트림 쓰레드가 예기치 않게 종료될 때 항상 호출된다.
``` java
streams.setUncaughtExceptionHandler((Thread thread, Throwable throwable) -> {
  // here you should examine the throwable/exception and perform an appropriate action!
});
```
어플리케이션이 종료될 땐 아래와 같이 인스턴스를 멈춘다.
``` java
streams.close();
```
운영체제에서 프로세스가 죽을때 운영체제가 보내는 SIGTERM 이라는 시그널이 있는데 이 시그널을 받았을 때도 우아한 종료(Graceful shutdown) 하기 위해서 셧다운 훅을 등록시키는 것을 권장한다.
``` java
Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
```

## Streams application 설정하기
카프카는 항상 시작시키기 전에 설정을 해야 한다. StreamsConfig 라는 객체를 만들면 쉽게 설정 가능하다.
``` java
Properties prop = new Properties();
prop.put(StreamsConfig.APPLICATION_ID_CONFIG, "my-stream");
prop.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-broker1:9092");
...
```

## Streams DSL
(Domain Specific Language)
* Streams DSL 로는  KStream, KTable, GlobalKTable 를 만들 수 있다. 
	* Stream 과 Table 에대한 지원이 중요한 이유는 대부분의 사용 케이스에서 스트림 또는 테이블 각기 사용되기 보단 둘다 조합해서 사용하는 것이 필요하기 때문이다. 
* map, filter 와 같은 함수형 프로그래밍 스타일의 **상태가 없는** 트랜스포메이션( Stateless Transformation) 가능.
* Count, reduce, aggregation, join, windowing 같은  **상태가 있는** 트랜스포메이션(Stateful Transformation)도 가능하다.

DSL 을 사용하면 어플리케이션에서 Processor topologies(= logical processing plan) 을 정의할 수 있다. 아래와 같이 정의하면 된다.
1. 카프카 토픽을 읽어오는 하나 이상의 인풋 스트림을 정의하라. (Source)
2. 이런 스트림에 변형을 구성해라.
3. 카프카 토픽으로 다시 전달하는 아웃풋 스트림을 정의해라. 또는 결과 데이터를 직접 다른 어플리케이션으로 보내도 좋다. (Sink)

어플리케이션이 한번 동작하면 정의된 토폴로지가 단계별로 실행된다. 


### source stream 작성하기

#### KStream
``` java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Long> wordCounts = builder.stream(
    "word-counts-input-topic", /* input topic */
    Consumed.with(
      Serdes.String(), /* key serde */
      Serdes.Long()   /* value serde */
    );
```
serdes 의 경우 스트림의 컨피그 값으로 넣어주지 않았거나 컨피그 값으로 넣어준 serilalizer, deserializer와 다른 경우 명시한다.
만약에 많은 토픽으로부터 한번에 읽고싶다면 정규화식으로 input topic 부분에 넣어줘도 가능하다. 이런식으로 여러가지 변형된 stream 메소드가 존재하므로 참고.

#### KTable
토픽을 KTable 로 읽게되면 그 토픽은 `changelog` 스트림으로 번역되어진다.
> `ChangeLog` 로 번역되어지는 값은 아래와 같다.
>  * 값이 null 이 아닌경우
	  * 키가 같은 레코드가 있다면
		* Upsert가 되어졌다고 표현된다.
>  * 값이 null 인 경우
	  * Delete 되어졌다고 표현된다.
KTable 은 쿼리를 할 수 있는데, 만약에 KTable 을 만들 때 이름을 지정해 주지 않았다면 쿼리를 할 수 없다. 좀 더 정확하게 말하면 그 KTable 이 내용을 백업하는 내부적인 State Store 에게 이름을 주지 않았기 때문에 쿼리를 할 수 없는것. 
만약 이름을 지정하지 않으면 State Store는 내부적인 이름을 임의로 제공받는다.

KTable 역시 config 로 넣어준 serde 를 사용하고 명시된 경우 명시된 serde 를 사용한다. 
또한 여러가지 유형으 KTable 생성 메소드가 있으므로 참고. auto.offset.reset 하는 정책 설정도 가능.

#### GlobalKTable
globalKTable 의 경우에도 생성하면 `changeLog` 스트림이 생긴다. changelog 로 번역되어지는 값은 KTable 과 같다.
globalKTable 의 경우는 모든 어플리케이션 인스턴스의 로컬 GlobalKTable 인스턴스가 **카프카 토픽의 모든 파티션** 에서 온다. (KStream, KTable 의 경우는 어플리케이션의 인스턴스가 가지는 로컬 KStream, KTable 값은 분배된 토픽 파티션 값만 가지고 있다)

GlobalKTable 도 쿼리하기 위해서는 이름을 명시해 줘야 한다. 이경우 에도 테이블 값이 백업되는 State Store의 이름이 명시되어야 쿼리가 가능하기 때문이다. 
``` java
StreamsBuilder builder = new StreamsBuilder();

GlobalKTable<String, Long> wordCounts = builder.globalTable(
    "word-counts-input-topic",
    Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as(
      "word-counts-global-store" /* table/store name */)
      .withKeySerde(Serdes.String()) /* key serde */
      .withValueSerde(Serdes.Long()) /* value serde */
    );
```
GlobalKTable 도 여러 형태의 생성 메소드가 있다. 

### Stream 에 변형 가하기
인풋 스트림에 변형을 가하면 그 변형 하나하나가 토폴로지에서 하나의 processor 로 여겨진다. 
filter, map 의 경우 새로운 스트림을 생성하게된다. 
branch 는 하나의 KStream을 여러개의 KStream 으로 만들어서 사용할 수 있게 한다. 또한 KStream 에서 KTable 로 변경시켜주는 오퍼레이션도 있다. 

모든 KTable 변경 동작은 다른 KTable 을 만드는 작업이다. 하지만 kafka streams DSL 은 KTable 를 KStream으로 표현 특별한 메소드를 제공한다. 

이런 모든 변형 동작은 체인형으로 사용할 수 있고 이 모든 동작의 조합이 토폴로지가 된다.

#### Stateless Transformations (no state store)
stateless transformation 은 처리를 위한 상태가 필요 없는 동작이고 또한 State Store 가 필요하지 않기 때문에 생성되지 않는다. 
상태가 없는 KTable 도 카프카 0.11.0 부터 지원하기 시작했는데 이를 위해서 materialize 를 제공한다. 이 materialized KTable 를 사용하면 쿼리가 가능하다.  
KTable 를 materialize 하기 위해서  queryableStoreName 파라미터를 줘서 상태가 없는 operations 들을 사용할 수 있다.

Transformation | Description
------ | ------
**Branch**  | 하나 또는 그 이상의 KStream 로 만드는데 predicates를 넣어 사용할 수 있다. 
KStream -> KStream[]| predicates 는 순서대로 평가되어지고 매칭되는 순간 해당 스트림으로 분류된다. (즉 먼저 매칭되는 것으로 분류됨) 매칭되는 predicates 가 없다면 그 데이터(레코드)는 버려진다.

``` java
 KStream<String, Long> stream = ...;
 KStream<String, Long>[] branches = stream.branch(
    (key, value) -> key.startsWith("A"), /* first predicate  */
    (key, value) -> key.startsWith("B"), /* second predicate */
    (key, value) -> true                 /* third predicate  */
  ); 
```

Transformation | Description
------ | ------
**Filter**  | boolean function 을 평가해서 true 가 되는 데이터들만 남긴다.
KStream → KStream | 
KTable → KTable| 

``` java
KStream<String, Long> onlyPositives = stream.filter((key, value) -> value > 0);
```

Transformation | Description
------ | ------
**FilterNot**  | boolean function 을 평가해서 true 가 되는 데이터들만 버려진다.
KStream → KStream | filter 의 반대
KTable → KTable| 

``` java
KStream<String, Long> onlyPositives = stream.filterNot((key, value) -> value <= 0);
```

Transformation | Description
------ | ------
**FlatMap**  | 하나의 레코드로 0개 이상의 레코드를 만들어낸다. 레코드의 Key, value 를 수정할 때 사용한다.
KStream → KStream | Re-Partitioning 을 위한 marking 을 한다. 

일반적으로 파티셔닝 알고리즘은 key의 hash 값을 사용하기 때문에 key가 바뀐다는 것은 re paritioning 을 한다는 의미이다. 그래서 flatMap 을 사용하는 순간 잠재적인 Re partitioning 을 위해 마킹을 하고 재분배를 위한 인터널 토픽을 생성하는 작업이 이루어진다. 그렇기 때문에 만약 value 만 바꿔야 하는 상황이라면 `FlatMapValues` 를 사용하는 것을 권장한다. 

``` java
KStream<String, Integer> transformed = stream.flatMap(
     // Here, we generate two output records for each input record.
     // We also change the key and value types.
     // Example: (345L, "Hello") -> ("HELLO", 1000), ("hello", 9000)
    (key, value) -> {
      List<KeyValue<String, Integer>> result = new LinkedList<>();
      result.add(KeyValue.pair(value.toUpperCase(), 1000));
      result.add(KeyValue.pair(value.toLowerCase(), 9000));
      return result;
    }
  );
```

Transformation | Description
------ | ------
**FlatMapValues**  | 하나의 레코드로 0개 이상의 레코드를 만들어낸다. 레코드의 value 만 수정할 때 사용한다.
KStream → KStream |


``` java
KStream<byte[], String> words = sentences.flatMapValues(value -> Arrays.asList(value.split("\\s+")));
```

Transformation | Description
------ | ------
**Foreach**  | 최종에 사용할 수 있는 operation 이다. 카프카에 영향을 미치지 않는다.
KStream → void | 사이드 이펙트 처럼 부가적으로 하는 작업일 때 사용하므로 최종에만 올수 있다.
KTable → void | `peek` 과 다르게 리턴값은 없다. 

``` java
stream.foreach((key, value) -> System.out.println(key + " => " + value));
```

Transformation | Description
------ | ------
**GroupByKey**  | key 별로 그룹을 만든다. aggregate 하기전에 해야 한다. 변형이 있어서 Serdes를 다르게 사용해야 한다면 재정의해야 한다.
KStream → KGroupedStream | re-partitioning 이 일어나지 않는다. key 값을 변경할 수 없다. 

연관된 operation 은 `windowing` 이다. windowing 은 windowed aggregation 이나 windowed joins 와 같은 *상태가 있는* operation 인데  같은 키의 레코드들을 윈도우로 그룹화 할 수 있다. 

> groupByKey 가 GroupBy 보다 권장되는데 그 이유는 GroupBy 의 경우 이미 리파티셔닝 되도록 마킹이 되어있을 경우 리파티셔닝이 일어난는데  GroupByKey 는 GroupBy와 다르게 key를 수정하지 못하기 때문이다.

``` java
KStream<byte[], String> stream = ...;

// Group by the existing key, using the application's configured
// default serdes for keys and values.
KGroupedStream<byte[], String> groupedStream = stream.groupByKey();

// When the key and/or value types do not match the configured
// default serdes, we must explicitly specify serdes.
KGroupedStream<byte[], String> groupedStream = stream.groupByKey(
    Serialized.with(
      Serdes.ByteArray(), /* key */
      Serdes.String())     /* value */
  );
```

Transformation | Description
------ | ------
**GroupBy**  | 다른 키가 될 수있는 새로운 키를 기준으로 그룹을 만든다. key, value를 바꿀 수 있다. 
KStream → KGroupedStream | aggregation 하기 위한 이전 작업으로 사용되고 이후 데이터가 적절히 파티션된다.
KTable → KGroupedTable | 이것 역시 key, value 가 변해서 serdes 가 변해야 한다면 명시해야한다.

GroupBy 는 늘 re-partitioning 을 야기한다. groupByKey 를 사용할 수 있으면 groupByKey를 사용하는 것을 권장한다. 

``` java
// Group the stream by a new key and key type
KGroupedStream<String, String> groupedStream = stream.groupBy(
    (key, value) -> value,
    Serialized.with(
      Serdes.String(), /* key (note: type was modified) */
      Serdes.String())  /* value */
  );

// Group the table by a new key and key type, and also modify the value and value type.
KGroupedTable<String, Integer> groupedTable = table.groupBy(
    (key, value) -> KeyValue.pair(value, value.length()),
    Serialized.with(
      Serdes.String(), /* key (note: type was modified) */
      Serdes.Integer()) /* value (note: type was modified) */
  );
```

Transformation | Description
------ | ------
**Map**  | 하나의 레코드로 하나의 레코드를 만드는데 key, value 를 수정할 수 있다.
KStream → KStream | re-partitioning 될 수 있으므로 가급적 MapValues 사용.

``` java
KStream<String, Integer> transformed = stream.map(
    (key, value) -> KeyValue.pair(value.toLowerCase(), value.length()));
```

Transformation | Description
------ | ------
**MapValues**  | 하나의 레코드로 하나의 레코드를 만드는데 value 만 수정할 수 있다.
KStream → KStream |
KTable → KTable |

``` java
KStream<byte[], String> uppercased = stream.mapValues(value -> value.toUpperCase());
```

Transformation | Description
------ | ------
**Peek**  | 각각의 레코드를 받아서 사이드 이펙트 작업을 할 수 있다. 리턴값은 **변경되지 않은** 스트림이다.
KStream → KStream | 로깅할때 사용 하거나 카프카에 영향을 미치지 않는 작업에 사용된다.

``` java
KStream<byte[], String> unmodifiedStream = stream.peek(
    (key, value) -> System.out.println("key=" + key + ", value=" + value));
```


Transformation | Description
------ | ------
**Print**  | 최종 operation 으로 사용가능하다.
KStream → void | key, value 를 system.out.println() 해 준다.

``` java
stream.print();
```

Transformation | Description
------ | ------
**SelectKey**  | key 값을 바꿀 때 사용한다. 
KStream → KStream | Re-partitioning 마킹 된다.
 
``` java
KStream<String, String> rekeyed = stream.selectKey((key, value) -> value.split(" ")[0])
```

Transformation | Description
------ | ------
**toStream**  | 해당 테이블의ㅣ changelog Stream 으로부터 KStream 을 얻는다.
KTable → KStream |

``` java
KStream<byte[], String> stream = table.toStream();
```


#### Stateful Transformations (State Store)
상태가 있는 변형은 인풋을 프로세싱하고 아웃을을 생성하기 위한 state 에 의존한다.  그리고 State Store 가 있어야한다.	
* windowing 의 State Store
	* 정해진 window 영역부터 들어온 레코드들을 모아서 가지고 있는데에 사용한다.
![Alt text](https://monosnap.com/image/ekjMsbQI1hRI7NSkoJNqM7im2eMX0m)

Transformation | Description
------ | ------
**Aggregate**  | KGroupedStream 이나 KGroupedTable 로 변형한 상태로 사용할 수 있다.
KGroupedStream → KTable | key 기반 동작 이다.
KGroupedTable → KTable |

KGroupedStream 을 aggregate 할 때는 initializer 와 **adder** 를 제공해야 하고
KGroupedTable 을 aggregate 할 때는 initializer 와 **subtractor** 를 제공해야 한다.

``` java
// Aggregating a KGroupedStream (note how the value type changes from String to Long)
KTable<byte[], Long> aggregatedStream = groupedStream.aggregate(
    () -> 0L, /* initializer */
    (aggKey, newValue, aggValue) -> aggValue + newValue.length(), /* adder */
    Materialized.as("aggregated-stream-store") /* state store name */
        .withValueSerde(Serdes.Long()); /* serde for aggregate value */

// Aggregating a KGroupedTable (note how the value type changes from String to Long)
KTable<byte[], Long> aggregatedTable = groupedTable.aggregate(
    () -> 0L, /* initializer */
    (aggKey, newValue, aggValue) -> aggValue + newValue.length(), /* adder */
    (aggKey, oldValue, aggValue) -> aggValue - oldValue.length(), /* subtractor */
    Materialized.as("aggregated-table-store") /* state store name */
	.withValueSerde(Serdes.Long()) /* serde for aggregate value */
```
> Aggregate 의 동작 과정
> * KGroupedStream 의 경우
> null 키를 가지는 레코드는 무시된다.
> 레코드 가 처음으로 도착하면 초기화(initializer) 가 호출된다.
>
> null value 가 아닌 레코드가 들어올 때마다 adder 가 호출된다. 
> ![Alt text](https://monosnap.com/image/MaHZlX3UvPNhzWrFqn3aZvHhyTMcM5)
> 
> * KGroupedTable 의 경우
> null 키를 가지는 레코드는 무시된다.
> 레코드 키가 처음으로 도착할때 초기화(initializer) 가 호출된다.
>
> KGroupedStream 와 다르게 thumstone 을 가지는 레코드가 있기 때문에 초기화 (initializer)가 여러번 호출 될 수 도 있다.
> 첫번째로 null 이 아닌 레코드가  들어오면 adder 가 호출된다.
> 바로 뒤에 null 이 아닌 레코드가 들어오면 
>
> (1) subtractor 가 old value 와 함께 호출되고  
> 
> (2) adder가 새롭게 들어온 레코드 값과 함께 호출된다. 실행 순서는 정해져있지 않다.
>
> thumstone 을 가진 레코드에 대해서는
>
> null 값을 가지면 subtractor 만 호출된다. -> 만약 연산 결과가 null 이면 KTable 에서 제거된다.
>
> 다음 인풋 레코드가 초기화된다.
> ![Alt text](https://monosnap.com/image/Djl9CV3kbYKm1IE6zwXTljNcVEEGRN)


Transformation | Description
------ | ------
**WindowedBy**  | window 별로 그룹화 된 키를 기준으로 레코드 값을 집계한다.
KGroupedStream → KTable | 

initializer, adder, window 가 주어져야 한다. 세션에 기반한 윈도윙을 할 떄는 추가적으로 session merger가 주어져야 한다.
TimeWindowedKStream<K, V> , SessionWindowedKStream<K, V>  를 할 수 있고 KTable 로 만들 수 있다.
``` java
KTable<Windowed<String>, Long> timeWindowedAggregatedStream = groupedStream.windowedBy(TimeUnit.MINUTES.toMillis(5))
    .aggregate(
      () -> 0L, /* initializer */
    	(aggKey, newValue, aggValue) -> aggValue + newValue, /* adder */
      Materialized.<String, Long, WindowStore<Bytes, byte[]>>as("time-windowed-aggregated-stream-store") /* state store name */
        .withValueSerde(Serdes.Long())); /* serde for aggregate value */

// Aggregating with session-based windowing (here: with an inactivity gap of 5 minutes)
KTable<Windowed<String>, Long> sessionizedAggregatedStream = groupedStream.windowedBy(SessionWindows.with(TimeUnit.MINUTES.toMillis(5)).
    aggregate(
    	() -> 0L, /* initializer */
    	(aggKey, newValue, aggValue) -> aggValue + newValue, /* adder */
    	(aggKey, leftAggValue, rightAggValue) -> leftAggValue + rightAggValue, /* session merger */
	    Materialized.<String, Long, SessionStore<Bytes, byte[]>>as("sessionized-aggregated-stream-store") /* state store name */
        .withValueSerde(Serdes.Long())); /* serde for aggregate value */

```
windowing 은 위의 aggregate 와 비슷하게 동작한다. 지정된 창에서 레코드가 처음으로 들어오면 초기화가 호출된다. 그 이후 레코드가 들어올 떄마다 adder 를 호출한다. session window 의 경우 두 세션이 병합될 때마다 세션 병합이 호출된다. 

Transformation | Description
------ | ------
**count**  | key 로 그룹된 레코드의 수를 센다.
KGroupedStream → KTable |
KGroupedTable → KTable | 

``` java
// Counting a KGroupedStream
KTable<String, Long> aggregatedStream = groupedStream.count();

// Counting a KGroupedTable
KTable<String, Long> aggregatedTable = groupedTable.count();
```

Transformation | Description
------ | ------
**count(window)**  | window 당 레코드의 수를 센다. 
KGroupedStream → KTable |

``` java
// Counting a KGroupedStream with time-based windowing (here: with 5-minute tumbling windows)
KTable<Windowed<String>, Long> aggregatedStream = groupedStream.windowedBy(
    TimeWindows.of(TimeUnit.MINUTES.toMillis(5))) /* time-based window */
    .count();

// Counting a KGroupedStream with session-based windowing (here: with 5-minute inactivity gaps)
KTable<Windowed<String>, Long> aggregatedStream = groupedStream.windowedBy(
    SessionWindows.with(TimeUnit.MINUTES.toMillis(5))) /* session window */
    .count();
```

Transformation | Description
------ | ------
**reduce**  | 그룹화된 키를 기준으로 레코드를 결합한다.aggregate 와 다르게 value 타입은 변경할 수 없다.
KGroupedStream → KTable |
KGroupedTable → KTable |

``` java
// Reducing a KGroupedStream
KTable<String, Long> aggregatedStream = groupedStream.reduce(
    (aggValue, newValue) -> aggValue + newValue /* adder */);

// Reducing a KGroupedTable
KTable<String, Long> aggregatedTable = groupedTable.reduce(
    (aggValue, newValue) -> aggValue + newValue, /* adder */
    (aggValue, oldValue) -> aggValue - oldValue /* subtractor */);
```


Transformation | Description
------ | ------
**Reduce(window)**  | 윈도우 별로 레코드를 결합한다. 현재 레코드의 값이 마지막 벨류와 결합된다. aggregate와 다르게 타입 변경이 불가능하다.
KGroupedStream → KTable |

``` java
// Aggregating with time-based windowing (here: with 5-minute tumbling windows)
KTable<Windowed<String>, Long> timeWindowedAggregatedStream = groupedStream.windowedBy(
  TimeWindows.of(TimeUnit.MINUTES.toMillis(5)) /* time-based window */)
  .reduce(
    (aggValue, newValue) -> aggValue + newValue /* adder */
  );

// Aggregating with session-based windowing (here: with an inactivity gap of 5 minutes)
KTable<Windowed<String>, Long> sessionzedAggregatedStream = groupedStream.windowedBy(
  SessionWindows.with(TimeUnit.MINUTES.toMillis(5))) /* session window */
  .reduce(
    (aggValue, newValue) -> aggValue + newValue /* adder */
  );
```
#### Join
join 으로 다양한 데이터를 합칠 수 있다.
![Alt text](https://monosnap.com/image/INWNod4kjm8oom7E3j4nGapm4GzSmr)

조인하기 위한  조건이 있는데 co-partitioning 해야 한다는 것이다.
##### co-partitioning 이란 
같은 키값을 가지는 인풋 레코드가 join 의 양쪽 에 다 존재해야 join 할 수 있기 때문에 파티션 개수를 동일하게 맞춰야 한다.
대신 **GlobalKTable**의 경우는 파티션 갯수와 무관하다. 모든 데이터가 한 어플리케이션 인스턴스에 존재하기 떄문이다.
인풋 토픽에 write 하는 모든 어플리케이션은 같은 파티셔닝 전략을 가지고 있어야한다. 그래야 같은 키가 같은 파티션 번호를 가지고 전달되어지므로. 즉, 동일한 인스턴스로 분산되어야 된다는 뜻이다. 기본 파티셔닝 전략을 바꾸지 않았다면 신경쓰지 않아도 된다.

#### Windowing
윈도우로 그룹핑하는 것을 말한다.
조인 동작에서 윈도우 바운더리에 들어오는 레코드들을 저장하는데 State Store가 사용되어진다.
집계동작에서 윈도우 당 가장 나중의 집계가 저장되는데에 State Store 가 사용되어진다. State Store 의 오래된 레코드들은 window retention period 가 지나면 제거 된다.
카프카 스트림은 최소한 이 주어진 시간동안 윈도우를 유지할 수 있도록 보장하고 기본값은 **하루**이고 이 값은 until() 로 바꿀 수 있다. 


Window name |	Behavior	| Short description
----- | ----- | ------ 
Tumbling time window	|Time-based |	고정 사이즈, 오버랩 없음, 갭이 적은 windows
Hopping time window	| Time-based	| 고정 사이즈, 오버랩되는 windows
Sliding time window|	Time-based |	고정 사이즈, 레코드의 타임스탭프 사이의 차이로 동작하는 오버랩 되는 windows
Session window |	Session-based	| 유동적 사이즈, 오버랩 없음, data-driven windows


##### Tumbling time windows (회전 시간 윈도우)
![Alt text](https://monosnap.com/image/eTL7ko0zxmldZWtfBDRAkX1SOyLV0z)

덤블링 타임 윈도우는 시간 간격에 기반한 윈도우다. 그림과 같이 사이즈는 고정이고 겹치지 않으며 갭이 적은 윈도우다. 윈도우 사이즈만 정해주면 정의할 수 있다. 덤블링 윈도우는 hopping 윈도우인데 오버랩이 되지 않기때문에 데이터 레코드는 단 하나의 윈도우에만 속하게 된다. 분 단위로 작성이 가능하기 때문에 5분일 경우 아래와 같이 작성한다.
``` java
long windowSizeMs = TimeUnit.MINUTES.toMillis(5); 
TimeWindows.of(windowSizeMs);

아래는 위와 같은 의미의 코드다. 
TimeWindows.of(windowSizeMs).advanceBy(windowSizeMs);
```
>NOTE: 시간이 포함되는 범위를 좀 더 상세히 얘기하자면, 
>시간 범위에서 예를 들어 5000 ~ 10000 ms 일 경우 시작 범위인 5000 은 `포함`이고 끝 범위인 10000은 `미포함` 이다. 즉 `시작 이상 끝 미만` 이다. 그리고 윈도우의 시작은 타임스탬프의 0ms 부터 시작이다. 
>그래서 만약에  설정해준 interval이 5000 이었다면 1000 ~ 6000 과 같은 범위는 절대 생길 수 없다. 0부터 시작하기 때문에 0~5000, 5000~10000 단위로 지나가기 때문이다.

##### Hopping time window 
hopping time window 는 시간 interval 를 기반으로 한 윈도우다. 
![Alt text](https://monosnap.com/image/PjWJ3211Cu0YbvZFBR20zxz3BHEBJi)
이런 hopping time window 들은 공통적으로 `고정된 사이즈` 이고 `오버랩 가능성`이 있는 윈도우다. 물론 덤블링 타임 윈도우처럼 오버랩을 사용하지 않는 윈도우도 있다. hopping time window 는 두개 속성으로 정의하는데 `윈도우 사이즈`와 `advanced interval`(=interval, =hop) 이다. advanced interval 은 이전에 비해 얼마나 윈도우가 앞으로 이동을 하느냐를 말한다. 예를 들면 사이즈가 5 분이고 advanced interval 이 1분이면 오버랩이 된다. 만약 사이즈 5분인 윈도우가 5분의 advanced interval 을 갖는다면 5분 전진 이기 때문에 오버랩이 전혀 없는 덤블링 타임 윈도우가 된다. 
``` java
long windowSizeMs = TimeUnit.MINUTES.toMillis(5); // 5 * 60 * 1000L
long advanceMs =    TimeUnit.MINUTES.toMillis(1); // 1 * 60 * 1000L
TimeWindows.of(windowSizeMs).advanceBy(advanceMs);
```
> NOTE: hopping time window vs sliding window
> hopping time window 는 슬라이딩 윈도우와 다르다. 

윈도잉이 아닌 aggregate 와 다르게 window aggregation은 키가 `windowed<K>` 인 windowed KTable 을 리턴한다. `ex.  KTable<Windowed<String>, Long>`  만약 window를 얻으려면 (k, v) -> k.window() 하면 되고 key 를 얻으려면 (k, v) -> k.key() 하면 된다. 

##### Sliding time windows
슬라이딩은 hopping이나 덤블링이랑 좀 다른데 카프카 스트림에서 슬라이딩 윈도우는 `join operation` 에서만 사용된다. 슬라이딩 윈도우는 시간 축 위에서 연속적으로 미끄러지는 고정 크기의 window 다. 덤블링 타임윈도우와 hopping 윈도우와는 다르게 시작 지점 이상 ~ 끝 지점 이하 다. 즉 `시작 끝 모두 포함`이다.

##### Session windows
session window 는 키 기반 이벤트를 '세션' 이라고 불리는 프로세스로 aggregate 하는데 사용되어진다. 세션은 유휴기간 또는 비활동의 갭으로 정의되는 `활동 기간` 을 의미한다. 세션의 비활성 간격 내에 있는 모든 이벤트들은 처리되어지고 세션에 머지된다. 만약에 세션 갭 밖에 이벤트가 존재하게 되면 새로운 세션이 생긴다. 
![Alt text](https://monosnap.com/image/GUsmShNPgMG9YsiQU5zl206fggolHM)
세션 윈도우는 다른 윈도우와 좀 다른데 
* 모든 윈도우는 키에 따라 독립적으로 추적된다. 예를 들면 키가 다른 윈도우는 일반적으로 다른 시작과 끝 시간을 가진다. 
* 윈도우 사이즈는 다양하다. 심지어 같은 키를 가진 윈도우도 다른 사이즈를 가진다. 

세션 윈도우를 위한 어플리케이션 영역은 사용자 동작 분석이다. 세션 기반의 분석은 간단한 metrics 로부터 (뉴스 사이트 방문횟수) 복잡한 metrics (이벤트 흐름) 까지 다양하다.
비활동 갭을 5분으로 주는 예제이다.
``` java
SessionWindows.with(TimeUnit.MINUTES.toMillis(5));
```


### processor API, transformers API 적용하기 
* 커스터마이징: 특별하고 커스텀 로직을 붙일 수 있다.
* 더 많은 유연성과 마음대로 주무르는것이 가능하다. 예를 들면 토픽, 파티션, 오프셋 정보와 같은 레코드의 metadata 에 접근할 수도 있다.
* 다른 툴과의 마이그레이션: 다른 스트리밍 기술과 마이그레이션이 더 쉽고 빠르게 가능하다

Transformation | Description
------ | ------
**Process**  | 최종 연산자로 사용 가능하고 각 레코드에 Processor를 적용하게 해준다. process()를 통해 processor API 를 사용할 수 있다.
KStream → void | topoloy 에 addProcessor() 하는 것과 같은 기능이다.

Transformation | Description
------ | ------
**Transform**  | 각각의 레코드에 Transformer 를 적용할 수 있다. transform() 을 통해 Processor API 를 사용할 수 있다.
KStream -> KStream | 0 이상의 아웃풋 레코드를 만들어 낼 수 있다. (flatMap과 비슷) 아웃풋을 없애려면 null 을 리턴하면 된다. 레코드의 key, value 를 수정할 수 있다. 그러므로 re-partitioning 으로 마킹이 된다. transformValues() 가 권장됨.

Topoloy 에 addProcessor() 를 하고 나서 transformer 를 추가시킨것과 같은 동작을 한다.

Transformation | Description
------ | ------
**TransformValues**  | 각각의 레코드에 ValueTransformer 를 적용할 수 있다. Processor API 를 사용할 수 있도록 하고 각각의 인풋 레코드는 무조건 하나의 아웃풋 레코드로 나온다.
KStream → KStream | Topoloy 에 addProcessor() 를 하고 나서 valueTransformer 를 추가시킨것과 같은 동작을 한다.


Processor 만들기 예제
``` java
public class PopularPageEmailAlert implements Processor<PageId, Long> {

  private final String emailAddress;
  private ProcessorContext context;

  public PopularPageEmailAlert(String emailAddress) {
    this.emailAddress = emailAddress;
  }

  @Override
  public void init(ProcessorContext context) {
    this.context = context;

    // Here you would perform any additional initializations such as setting up an email client.
  }

  @Override
  void process(PageId pageId, Long count) {
    // Here you would format and send the alert email.
    //
    // In this specific example, you would be able to include information about the page's ID and its view count
    // (because the class implements `Processor<PageId, Long>`).
  }

  @Override
  void close() {
    // Any code for clean up would go here.  This processor instance will not be used again after this call.
  }

}
```

``` java
KStream<String, GenericRecord> pageViews = ...;

// Send an email notification when the view count of a page reaches one thousand.
pageViews.groupByKey()
         .count()
         .filter((PageId pageId, Long viewCount) -> viewCount == 1000)
         // PopularPageEmailAlert is your custom processor that implements the
         // `Processor` interface, see further down below.
         .process(() -> new PopularPageEmailAlert("alerts@yourcompany.com"));
```

### streams 카프카로 다시 쓰기

Transformation | Description
------ | ------
**To**  | 최종 연산자, 레코드를 다시 카프카 토픽에 쓴다. 기존에 쓰던 Serdes 와 다른것을 써야한다면 명시한다.
KStream → void | 커스텀한 파티셔닝 전략이 필요한 경우 StreamPartitioner 를 명시할 수 있다.

다음 중 하나일때 Re-Partitioning 이 일어난다. 
1. 아웃풋 토픽이 stream/table 과 다른 파티션갯수를 가지고 있을 경우
2. KStream 이 re-partitioning 으로 마킹되었을 경우
3. 제공된 StreamPartitioner 가 있을경우
4. 아웃풋 레코드의 키가 null 일 경우

Transformation | Description
------ | ------
**Through**  | 레코드를 카프카 토픽에 쓰고 해당 토픽에 대해 다시 새로운 스트림/테이블을 생성한다. serdes 가 맞지 않을경우 명시해야 한다. 
KStream -> KStream | 이 역시 StreamPartitioner를 명시할 수 있다. 
KTable -> KTable |

다음 중 하나일때 Re-Partitioning 이 일어난다. 
1. 아웃풋 토픽이 stream/table 과 다른 파티션갯수를 가지고 있을 경우
2. KStream 이 re-partitioning 으로 마킹되었을 경우
3. 제공된 StreamPartitioner 가 있을경우
4. 아웃풋 레코드의 키가 null 일 경우

## Processor API
개발자가 커스텀 Processor 를 연결하게 하고 state store와 상호작용을 할 수 있도록 해주는 api 다.  이 api 를 사용하면 임의의 Processor 를 구현할 수 있게 해주고 관련된 state store를 연결시키게 해준다.
Stateless, Stateful 하게 모두 사용이 가능하다. state store 를 사용하면 stateful 한 것.
위에서 설명한것 처럼 편하게 DSL 이랑 같이 사용하는 것이 가능하다. 
Processor 인터페이스를 구현해서 사용하는데 init() 은 카프카 스트림 라이브러리가 Task 생성시 호출한다. 기본 init() 동작은 ProcessorContext 를 건네는 작업이다. 이 context 안에는 현재 처리되는 레코드의 메타데이터(소스토픽, 파티션, 메세지 오프셋 등의 정보를 가지는)를 가진다. 이 context 는 schedule(), forword(), commit() 등을 처리할 수 있는데 특히나 schedule()의 경우는 PunctuationType 에 따라 주기적으로 punctuate()를 트리거하는 유저의 Punctuator 콜백 인터페이스를 허용한다. PunctuationType 은  punctuation 스케줄링에 사용되는 시간 개념, 기준을 잡는데 사용되어지는데 - 이전에 언급했던 그 wall-clock-time 또는 stream-time 중 하나를 말함. 디폴트는 TimestampExtractor 를 통해 이벤트 타임을 뜻하는 stream-time 이다-  stream-time을 사용할 때는 인풋 데이터에서 가져온 타임스템프에의해 결정되므로 데이터에 의해서만 punctuate()가 트리거링된다. 

예를 들어 60개의 레코드를 가진 스트림을 처리하는데 Punctuator 함수를 매 10초마다 PunctuationType.STREAM_TIME 로 스케줄링 했고 각 레코드는 1초부터 60초 까지 연속적인 타임스템프를 가지고 있다면 puntuate() 메소드는 6번 불릴 것이다. 실제 이 레코드들을 처리하는 시간이 1초든 1시간이든 상관 없이 발생한다. 대신 이 경우 새로운데이터가 들어오지 않으면 puntuate()는 호출되지 않는다.

또한 만약 PunctuationType.WALL_CLOCK_TIME 을 사용했을 경우 punctuate()는 wall-clock-time 에 맞춰 트리거링 된다. 60개의 레코드가 20초 안에 프로세싱 된다면 puntuate()는 2번 불릴 것이다. 만약에 5초 안에 60개의 레코드가 다 처리되면 puntuate()는 단 한번도 호출되지 않는다. 

schedule 예제
``` java
public class WordCountProcessor implements Processor<String, String> {

  private ProcessorContext context;
  private KeyValueStore<String, Long> kvStore;

  @Override
  @SuppressWarnings("unchecked")
  public void init(ProcessorContext context) {
      // keep the processor context locally because we need it in punctuate() and commit()
      this.context = context;

      // retrieve the key-value store named "Counts"
      kvStore = (KeyValueStore) context.getStateStore("Counts");

      // schedule a punctuate() method every 1000 milliseconds based on stream-time
      this.context.schedule(1000, PunctuationType.STREAM_TIME, (timestamp) -> {
          KeyValueIterator<String, Long> iter = this.kvStore.all();
          while (iter.hasNext()) {
              KeyValue<String, Long> entry = iter.next();
              context.forward(entry.key, entry.value.toString());
          }
          iter.close();

          // commit the current processing progress
          context.commit();
      });
  }

  @Override
  public void punctuate(long timestamp) {
      // this method is deprecated and should not be used anymore
  }

  @Override
  public void close() {
      // close any resources managed by this processor
      // Note: Do not close any StateStores as these are managed by the library
  }

}
```

### State Store
stateful Processor, Stateful Transformer 를 구현하려면 하나 이상의 state store 를 제공해야 한다. 
 State store 는 아래와같은 용도로 사용되어질 수 있다.
1. 최근에 전달받은 인풋 레코드를 저장하는데 사용되어질 수 있다.
2. rolloing aggregation 을 트래킹 하는데도 사용될 수 있다. 
3. 인풋 레코드 중복방지에도 사용 된다.
4. 다른 어플리케이션(node, scala, go etc...)과 상호작용할 수 있는 쿼리 기능이 가능하다.
4. 등...

### State Store 정의하고 사용하기

사용가능하도록 주어진 state store 중 하나를 사용해도 좋고 커스텀 스토어 타입을 지정해도 좋다. 한가지 중요한 점은 State Store 를 사용할때 인스턴스를 일반적으로 생성하면 안되고 StoreBuilder 를 통해 간접적으로 정의해야 한다. 

#### 영속적인 KeyValueStore<K, V>
storage engine 은 `RocksDB` 고 `내결함성`을 가지고 있다.
* 일반적인 경우에 이 종류의 state store 를 사용하는 것을 권장한다.
* 로컬 Disk 에 저장된다.
* 저장 용량은 어플리케이션의 메모리(힙 영역) 보다 커도 가능하지만 가능한 로컬 disk 공간에는 맞아야 한다.
* RocksDB 설정은 잘 튜닝 되어야 한다.
* window 의 key-value, session window의 key-value 도 저장가능
* 아래는 RocksDB 설정 예시

``` java
public static class CustomRocksDBConfig implements RocksDBConfigSetter {

       @Override
       public void setConfig(final String storeName, final Options options, final Map<String, Object> configs) {
         // See #1 below.
         BlockBasedTableConfig tableConfig = new org.rocksdb.BlockBasedTableConfig();
         tableConfig.setBlockCacheSize(16 * 1024 * 1024L);
         // See #2 below.
         tableConfig.setBlockSize(16 * 1024L);
         // See #3 below.
         tableConfig.setCacheIndexAndFilterBlocks(true);
         options.setTableFormatConfig(tableConfig);
         // See #4 below.
         options.setMaxWriteBufferNumber(2);
       }
    }

Properties streamsSettings = new Properties();
streamsConfig.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG, CustomRocksDBConfig.class);
```

영속적인 state store 생성하기 예제

``` java
StoreBuilder<KeyValueStore<String, Long>> countStoreSupplier =
  Stores.keyValueStoreBuilder(
    Stores.persistentKeyValueStore("persistent-counts"), // 이름
    Serdes.String(),
    Serdes.Long());
KeyValueStore<String, Long> countStore = countStoreSupplier.build();
```

#### 인 메모리 KeyValueStore<K, V>
기본으로 `내결함성`을 가진다.
* 인 메모리에 데이터를 저장한다.
* 저장 용량은 무조건 어플리케이션의 메모리 사이즈(힙 영역)에 맞아야 한다.
* 어플리케이션이 로컬 disk 영역을 사용할 수 없는 환경이거나 앱 인스턴스가 재시작하는 사이에 지워져야 하는 경우 유용하다.

``` java
StoreBuilder<KeyValueStore<String, Long>> countStoreSupplier =
  Stores.keyValueStoreBuilder(
    Stores.inMemoryKeyValueStore("inmemory-counts"), // 이름
    Serdes.String(),
    Serdes.Long());
KeyValueStore<String, Long> countStore = countStoreSupplier.build();
```

### 내 결함성 State Store
내결함성을 갖는 State Store 를 만들기 위해, 그리고 State Store 마이그레이션에서 데이터 누락을 배제하기 위해서 State Store 는 지속적으로 카프카 토픽에 비하인드 씬으로 백업을 한다. 이 토픽은 가끔 State Store 와 연관된 Changelog topic또는 changelog 를 참조한다. 예를 들어 시스템 오류가 발생하면 changelog 에서 완전히 복원시킬 수 있다. 원한다면 이 백업 토픽에 백업하는 기능을 끌 수도 있다. 영속적인 state store는 컴팩트 된 changelog 토픽에 의해서 백업되어진다. 

이 토픽이 compact되어지는 이유는 아래와 같다.
1. 이 토픽이 무한정 확장되는것을 방지하기 위함이다. 
2. 연관된 카프카 클러스터에서 consumed 되는것을 줄이기위해서다. 
3. State Store 가 그 changelog 토픽으로 부터 복구되어야 한다면 복구 시간도 줄일 수 있다. 

비슷하게 영속적인 window 저장소도 내결함성을 가진다. compacted 토픽에 백업을 한다. changelog 토픽으로 전송되는 메세지 키의 구조 때문에 삭제 및 압축이 필요하다. window Store에는 메세지 키가 일반 키와 윈도우 타임스템프의 결합된 구조로 들어간다. 이런 유형의 경우에는 이 결합된 복합키가 영역 을 넘어가면 안되기 때문에 compaction 뿐 아니라 deletion 도 사용된다. deletion 이 사용가능하므로 오래된 만기의 window는 카프카의 로그 클리너에 의해 삭제되어진다. 기본 retention 세팅은 Windows.maintainMs() + 1 일 이다. (이건 설정 StreamsConfig.WINDOW_STORE_CHANGE_LOG_ADDITIONAL_RETENTION_MS_CONFIG 변경 가능함)
한번 iterator 로 State Store 를 꺼냈으면 작업이 끝난 후 close() 를 호출해야 한다. 또는 try-with-resources 문장안에서 실행해야 한다. 이 작업을 해주지 않으면 OOM 에러.

### State Store 의 내 결함성 enable/disable (= changelog 저장 enable/disable)

changelog 를 disable 하면서 State Store 의 내결함성 을 키고 끌수도 있는데 withEnableLoggin() withDisableLoggin() 으로 가능하다. 
``` java
StoreBuilder<KeyValueStore<String, Long>> countStoreSupplier = Stores.keyValueStoreBuilder(
  Stores.persistentKeyValueStore("Counts"),
    Serdes.String(),
    Serdes.Long())
  .withLoggingDisabled(); // disable backing up the store to a changelog topic
```
만약 changelog 가 disable 되었으면 State Store 는 더이상 내결함성을 가지지 않고 어떠한 `standby replica` 도 가지지 않는다. 
> NOTE: Standby Replica 란?
>
> 대기중인 복제본을 말한다. 로컬 State Store 의 복사본인데 카프카 스트림은 config 에 설정된 지정된 수의 복제본을 작성하고 최신상태로 유지한다. 이 standBy Relica 는 시스템 오류로 state store 가 복구되어야 하는 경우 그 복구 지연 시간을 최소화 하는데 사용되어진다. 오류가 발생한 인스턴스에서 이전에 실행중이던 Task 는 standby replica 가 있는 인스턴스에서 재시작 되고 로컬 State Store 를 복구하는 프로세스가 최소화 된다. 

changelog 를 enable 하는 예제이다. 추가적으로 changelog 토픽에 대한 config 도 해주는 코드이다.

``` java
Map<String, String> changelogConfig = new HashMap();
// override min.insync.replicas
changelogConfig.put("min.insyc.replicas", "1")

StoreBuilder<KeyValueStore<String, Long>> countStoreSupplier = Stores.keyValueStoreBuilder(
  Stores.persistentKeyValueStore("Counts"),
    Serdes.String(),
    Serdes.Long())
  .withLoggingEnabled(changlogConfig); // enable changelogging, with custom changelog settings
```

### Custom State Store 만들기
org.apache.kafka.streams.processor.StateStore 인터페이스를 구현해야 한다. 카프카 스트림은 KeyValueStore 와 같은 store 를 이 인터페이스를 구현하여 작성하였다. 또한 org.apache.kafka.streams.state.StoreBuilder 를 구현하여 빌더도 제공해 줘야 한다.

### Processor와 State Store 연결하기
다음은 Processor API 로 (DSL 방식이 아니고) custom Processor와 state store 를 등록하는 과정이다. 
``` java
Topology builder = new Topology();

// add the source processor node that takes Kafka topic "source-topic" as input
builder.addSource("Source", "source-topic")

    // add the WordCountProcessor node which takes the source processor as its upstream processor
    .addProcessor("Process", () -> new WordCountProcessor(), "Source")

    // add the count store associated with the WordCountProcessor processor
    .addStateStore(countStoreBuilder, "Process")

    // add the sink processor node that takes Kafka topic "sink-topic" as output
    // and the WordCountProcessor node as its upstream processor
    .addSink("Sink", "sink-topic", "Process");
```
"Process" 를 붙인 스트림 프로세서는 "Source" 노드의 하류 프로세서로 인식되어지고 "Sink" 의 상류 프로세서로 인식되도록 한다. WordCountProcessor 의 Process() 메소드가 state store 를 트리거링 하고 context.forward() 할 때마다 aggregate 된 key-value 쌍이 sink 토픽으로 전송되어진다.  이전에 언급했던것과 마찬가지로 State Store 에 접근하려면 이름을 정의해 줘야 한다. 그렇지 않으면 state store 를 찾지 못해서 Runtime error 가 난다. 만약에 state store 가 토폴로지에서 연관되어있지 않다면.(정의되지 않았다면) WordCountProcessor의 init 메소드에서 Runtime error 가 발생한다. 

## Interactive Queries

interactive queries 는 다른 어플리케이션에서도 해당 어플리케이션의 State 를 사용할 수 있도록 한다. 카프카 스트림이 어플리케이션을 queryable 하게 만든다. 

![Alt text](https://monosnap.com/image/Hg0gvrKKTUWjQuFAfYcKo9WORlBnSI)
일반적으로 여러개의 인스턴스로 나눠져 있는 해당 어플리케이션의 전체 State 는 각 인스턴스 로컬에서 관리되는 여러개의 State Store 를 거쳐 존재한다. 
local 스토어가 있고 , 쿼리 할수 있는 remote 컴포넌트도 있다. 
* Local State
	* 어플리케이션 인스턴스가 내부적으로만 상태의 부분을 쿼리할 수 있고 직접 자신의 로컬 스토어를 쿼리한다. 
	* 카프카 스트림 api 가 필요하지 않는 한 해당 어플리케이션에서 쿼리한 로컬 데이터를 사용할 수 있다. 
	* State Store 는 항상 read-only 다.  add  작업도 불가능.
	* 항상 State store 는 작동하는 입력 데이터와 해당 프로세서 토폴로지에 의해서만 변형되어야 한다. 
	
* Remote State
	* 모든 State 를 쿼리하고 싶으면 State 의 많은 조각들을 연결시켜야 한다.
		* 로컬 State Store 를 쿼리할 수 있도록 해야하고 - 지원함
		* 모든 동작하는 인스턴스를 찾아야 하고 - 지원함
		* 이 인스턴스 들과 네트워크를 통해 통신해야한다. - 내부통신만 지원되므로 config 해야 한다!!
	* 이런 조각들을 연결시키는 작업은 인스턴스들 간의 통신을 하도록 만들고 다른 어플리케이션 간의 통신도 만들게 된다. 

카프카 스트림은 대화형 쿼리 (interactive queries) 를 통해 어플리케이션의 전체 상태를 표시하려는 경우를 제외하고 가능한 모든 기능을 제공한다. 어플리케이션 인스턴스가 네트워크를 통해 통신할 수 있도록 하려면 RPC 계층을 추가해야 한다.  Rest API 같은 것 추가.

### 하나의 앱 인스턴스를 위해서 Local State Store 쿼리하기
모든 어플리케이션 인스턴스는 로컬에 있는 모든 State Store 에 쿼리를 할 수 있는데 이름과 type 으로 쿼리 할 수 있다. 이름은 처음 만들어 줬을 때 지정해준 이름이고 type 은 keyValueStore() 또는 WindowStore() 이다. (커스텀을 준 경우 커스텀 타입)
카프카 스트림은 스트림 파티션 마다 하나의 State Store 를 가지는데(meterialize) 즉 굉장히 많은 State Store 가 존재한다는 의미다. 
( 각 인스턴스는 카프카 토픽의 분리된 파티션을 할당받는데 그 파티션은 카프카 스트림즈에서 Task 단위로 받아들여지는데 Task 별로 토폴로지가 존재하는 셈 이니까 하나의 토폴로지가 돌때 1개의 State Store가 생긴다면 Task 당 하나의  State Store. 즉 파티션 당 하나의 State Store 를 가진다.)
API 를 사용하면 어떤 파티션에 데이터가 들어있는지 관계없이 쿼리할 수 있다.
다음은 State Store 를 만들고 쿼리하는 예시이다.

``` java
// Define the processing topology (here: WordCount)
KGroupedStream<String, String> groupedByWord = textLines
  .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
  .groupBy((key, word) -> word, Serialized.with(stringSerde, stringSerde));

// Create a key-value store named "CountsKeyValueStore" for the all-time word counts
groupedByWord.count(Materialized.<String, String, KeyValueStore<Bytes, byte[]>as("CountsKeyValueStore"));

// Start an instance of the topology
KafkaStreams streams = new KafkaStreams(builder, config);
streams.start();
```
``` java
// Get the key-value store CountsKeyValueStore
ReadOnlyKeyValueStore<String, Long> keyValueStore =
    streams.store("CountsKeyValueStore", QueryableStoreTypes.keyValueStore());

// Get value by key
System.out.println("count for hello:" + keyValueStore.get("hello"));

// Get the values for a range of keys available in this application instance
KeyValueIterator<String, Long> range = keyValueStore.range("all", "streams");
while (range.hasNext()) {
  KeyValue<String, Long> next = range.next();
  System.out.println("count for " + next.key + ": " + value);
}

// Get the values for all of the keys available in this application instance
KeyValueIterator<String, Long> range = keyValueStore.all();
while (range.hasNext()) {
  KeyValue<String, Long> next = range.next();
  System.out.println("count for " + next.key + ": " + value);
}
```

기본적으로 Stateless 한 연산도 meterialize 해서 StateStore 로 만들 수 있다.

``` java
// materialize the result of filtering corresponding to odd numbers
// the "queryableStoreName" can be subsequently queried.
KTable<String, Integer> oddCounts = numberLines.filter((region, count) -> (count % 2 != 0),
  Materialized.<String, Integer, KeyValueStore<Bytes, byte[]>as("queryableStoreName"));

// do not materialize the result of filtering corresponding to even numbers
// this means that these results will not be materialized and cannot be queried.
KTable<String, Integer> oddCounts = numberLines.filter((region, count) -> (count % 2 == 0));
```

### Querying - Local Window Store
Window Store에는 여러 윈도우에 키가 있을 수 있기 때문에 여러개의 결과가 있을 수 있다. (각 윈도우에 1개인건 보장)
아래는 window store 를 생성하고 조회하는 코드다.
``` java
// Define the processing topology (here: WordCount)
KGroupedStream<String, String> groupedByWord = textLines
  .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
  .groupBy((key, word) -> word, Serialized.with(stringSerde, stringSerde));

// Create a window state store named "CountsWindowStore" that contains the word counts for every minute
groupedByWord.windowedBy(TimeWindows.of(60000))
  .count(Materialized.<String, Long, WindowStore<Bytes, byte[]>as("CountsWindowStore"));
```

``` java
// Get the window store named "CountsWindowStore"
ReadOnlyWindowStore<String, Long> windowStore =
    streams.store("CountsWindowStore", QueryableStoreTypes.windowStore());

// Fetch values for the key "world" for all of the windows available in this application instance.
// To get *all* available windows we fetch windows from the beginning of time until now.
long timeFrom = 0; // beginning of time = oldest available
long timeTo = System.currentTimeMillis(); // now (in processing-time)
WindowStoreIterator<Long> iterator = windowStore.fetch("world", timeFrom, timeTo);
while (iterator.hasNext()) {
  KeyValue<Long, Long> next = iterator.next();
  long windowTimestamp = next.key;
  System.out.println("Count of 'world' @ time " + windowTimestamp + " is " + next.value);
}
```

### Querying - Local Custom State Store
쿼리하기 전에 커스텀 State Store 는 아래를 만족해야 한다.
* StateStore 인터페이스를 구현해야 한다.
* 스토어에서 가능한 연산자들을 구현해야 한다.
* StoreBuilder 의 구현체를 제공해야 한다.
* 읽기 전용 연산으로 제한해야 한다. State Store 가 수정되는 걸 막아야 한다.
그래서 아래와 같은 형태여야 한다.

``` java
public class MyCustomStore<K,V> implements StateStore, MyWriteableCustomStore<K,V> {
  // implementation of the actual store
}

// Read-write interface for MyCustomStore
public interface MyWriteableCustomStore<K,V> extends MyReadableCustomStore<K,V> {
  void write(K Key, V value);
}

// Read-only interface for MyCustomStore
public interface MyReadableCustomStore<K,V> {
  V read(K key);
}

public class MyCustomStoreBuilder implements StoreBuilder {
  // implementation of the supplier for MyCustomStore
}
```

쿼리할 수 있으려면 아래를 만족시켜야 한다.
* QueryableStoreType 인터페이스의 구현체를 제공한다
* 저장소의 모든 기본 인스턴스에 액세스 할 수 있고 쿼리에 사용되는 래퍼 클래스를 제공한다

``` java
public class MyCustomStoreType<K,V> implements QueryableStoreType<MyReadableCustomStore<K,V>> {

  // Only accept StateStores that are of type MyCustomStore
  public boolean accepts(final StateStore stateStore) {
    return stateStore instanceOf MyCustomStore;
  }

  public MyReadableCustomStore<K,V> create(final StateStoreProvider storeProvider, final String storeName) {
      return new MyCustomStoreTypeWrapper(storeProvider, storeName, this);
  }
}
```

wrapper 클래스는 각각의 카프카 스트림 어플리케이션 인스턴스가 여러개의 스트림 Task 를 처리하고있고 여러개의 로컬 State Store 를 관리하고 있기 때문에 필요하다. 래퍼 클래스는 이러한 복잡성을 숨기고 해당 상태 저장소의 모든 기본 로컬 인스턴스를 알지 않고도 이름으로 "논리적"상태 저장소를 쿼리 할 수 있게 한다. 

wrapper 클래스를 구현할 때 StateStoreProvider 인터페이스를 사용해서 Store 의 잠재적인 인스턴스들에 접근할 수 있다. `StateStoreProvider#stores(String storeName, QueryableStoreType<T> queryableStoreType)` 가 주어진 이름을 가지는 State Store 리스트를 반환한다. 
아래는 wrapper 클래스 예시다.
``` java
// We strongly recommended implementing a read-only interface
// to restrict usage of the store to safe read operations!
public class MyCustomStoreTypeWrapper<K,V> implements MyReadableCustomStore<K,V> {

  private final QueryableStoreType<MyReadableCustomStore<K, V>> customStoreType;
  private final String storeName;
  private final StateStoreProvider provider;

  public CustomStoreTypeWrapper(final StateStoreProvider provider,
                                final String storeName,
                                final QueryableStoreType<MyReadableCustomStore<K, V>> customStoreType) {

    // ... assign fields ...
  }

  // Implement a safe read method
  @Override
  public V read(final K key) {
    // Get all the stores with storeName and of customStoreType
    final List<MyReadableCustomStore<K, V>> stores = provider.getStores(storeName, customStoreType);
    // Try and find the value for the given key
    final Optional<V> value = stores.stream().filter(store -> store.read(key) != null).findFirst();
    // Return the value if it exists
    return value.orElse(null);
  }
}
```
결과적으로 아래와 같이 커스텀 Store 를 쿼리할 수 있다.

``` java
StreamsConfig config = ...;
Topology topology = ...;
ProcessorSupplier processorSuppler = ...;

// Create CustomStoreSupplier for store name the-custom-store
MyCustomStoreBuilder customStoreBuilder = new MyCustomStoreBuilder("the-custom-store") //...;
// Add the source topic
topology.addSource("input", "inputTopic");
// Add a custom processor that reads from the source topic
topology.addProcessor("the-processor", processorSupplier, "input");
// Connect your custom state store to the custom processor above
topology.addStateStore(customStoreBuilder, "the-processor");

KafkaStreams streams = new KafkaStreams(topology, config);
streams.start();

// Get access to the custom store
MyReadableCustomStore<String,String> store = streams.store("the-custom-store", new MyCustomStoreType<String,String>());
// Query the store
String value = store.read("key");
```

### 전체 어플리케이션에서 Remote State Store 에 쿼리하기
전체 앱에서 Remote State Store 에 쿼리하기 위해서는 어플리케이션의 전체 스테이트를 다른 어플리케이션에 노출시켜야 한다. 
아래와 같은 작업이 필요하다.
![Alt text](https://monosnap.com/image/c5XfuMjxhdnFQkuCWglfUrw4k7KvTX)
1. 어플리케이션에 RPC 계층 추가하기
	* Rest API, Thrift, custom protocol 등...  네트워크를 통해 상호작용할수 있도록 추가한다.
2. RPC endpoint 노출하기
	* 카프카 스트림의 설정(application.server) 을 통해서 어플리케이션 인스턴스들의 endpoint 를 노출시킨다.
	* application.server 가 host:port 를 정하고 이게 endPoint 가 된다.
	* 이 값은 인스턴스마다 다르다. 
	* 카프카 스트림이 `StreamsMetadata` 인스턴스를 통해서 State Store, 할당된 파티션에 대한 RPC 엔드포인트 정보를 추적한다. 
		* 만약 application.server 를 설정해주지 않았다면 StreamsMetadata 를 찾을 수 없을 것.
3. 원격 어플리케이션 인스턴스 찾기
	* 원격 어플리케이션 인스턴스와 그 인스턴스들의 State Store  를 찾고 각 어플리케이션의 로컬 State Store 를 쿼리 가능하도록 해라
	* 원격 어플리케이션 인스턴스는 자기가 그 정보가 없다고 판단하면 쿼리를 다른 인스턴스로 포워딩 할 수 있다. 


아래는 config 에 rpcEndpoint 를 추가하는 예시이다.
``` java
Properties props = new Properties();
// Set the unique RPC endpoint of this application instance through which it
// can be interactively queried.  In a real application, the value would most
// probably not be hardcoded but derived dynamically.
String rpcEndpoint = "host1:4460";
props.put(StreamsConfig.APPLICATION_SERVER_CONFIG, rpcEndpoint);
// ... further settings may follow here ...

StreamsConfig config = new StreamsConfig(props);
StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> textLines = builder.stream(stringSerde, stringSerde, "word-count-input");

final KGroupedStream<String, String> groupedByWord = textLines
    .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
    .groupBy((key, word) -> word, Serialized.with(stringSerde, stringSerde));

// This call to `count()` creates a state store named "word-count".
// The state store is discoverable and can be queried interactively.
groupedByWord.count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>as("word-count"));

// Start an instance of the topology
KafkaStreams streams = new KafkaStreams(builder, streamsConfiguration);
streams.start();

// Then, create and start the actual RPC service for remote access to this
// application instance's local state stores.
//
// This service should be started on the same host and port as defined above by
// the property `StreamsConfig.APPLICATION_SERVER_CONFIG`.  The example below is
// fictitious, but we provide end-to-end demo applications (such as KafkaMusicExample)
// that showcase how to implement such a service to get you started.
MyRPCService rpcService = ...;
rpcService.listenAt(rpcEndpoint);
```

아래는 StreamsMetadata 를 찾는 예시이다.

``` java
KafkaStreams streams = ...;
// Find all the locations of local instances of the state store named "word-count"
Collection<StreamsMetadata> wordCountHosts = streams.allMetadataForStore("word-count");

// For illustrative purposes, we assume using an HTTP client to talk to remote app instances.
HttpClient http = ...;

// Get the word count for word (aka key) 'alice': Approach 1
//
// We first find the one app instance that manages the count for 'alice' in its local state stores.
StreamsMetadata metadata = streams.metadataForKey("word-count", "alice", Serdes.String().serializer());
// Then, we query only that single app instance for the latest count of 'alice'.
// Note: The RPC URL shown below is fictitious and only serves to illustrate the idea.  Ultimately,
// the URL (or, in general, the method of communication) will depend on the RPC layer you opted to
// implement.  Again, we provide end-to-end demo applications (such as KafkaMusicExample) that showcase
// how to implement such an RPC layer.
Long result = http.getLong("http://" + metadata.host() + ":" + metadata.port() + "/word-count/alice");

// Get the word count for word (aka key) 'alice': Approach 2
//
// Alternatively, we could also choose (say) a brute-force approach where we query every app instance
// until we find the one that happens to know about 'alice'.
Optional<Long> result = streams.allMetadataForStore("word-count")
    .stream()
    .map(streamsMetadata -> {
        // Construct the (fictituous) full endpoint URL to query the current remote application instance
        String url = "http://" + streamsMetadata.host() + ":" + streamsMetadata.port() + "/word-count/alice";
        // Read and return the count for 'alice', if any.
        return http.getLong(url);
    })
    .filter(s -> s != null)
    .findFirst();
```

## Memory 관리
내부 캐싱이나 레코드 compacting 을 위해서 메모리 사이즈를 설정할수 있다. 이 캐싱은 State Store 에 쓰이거나 하류 프로세서로 전달되기 전에 일어난다. 레코드 캐시는 DSL 과 Processor API 에서 살짝 다르다.

### DSL 에서 레코드 캐시
하류의 Stateful 프로세서 노드에서 내부 State Store에 쓰여지기 전에 아웃풋 레코드를 인터널 캐싱과 compacting 이 일어난다.
또한 하류의 processor node 에 전달되기 전에 인터널 캐싱과 compacting 이 일어난다.

aggregation 할때 캐싱에 따라 단계가 좀 달라지는데  <K,V>: <A, 1>, <D, 5>, <A, 20>, <A, 300> 이런 레코드가 있을 때

* 캐싱을 사용하지 않을 경우: 키 A에 대해 결과 집계 표의 변경 사항을 나타내는 일련의 아웃풋 레코드가 나온다. 괄호(())는 변경 사항을 나타내고, 왼쪽 번호는 새 집계 값이고 오른쪽 번호는 이전 집계 값 value: <A, (1, null)>, <A, (21, 1)>, <A, (321, 21)>.입니다. 

* 캐싱 시: 키 A에 대해 압축될 가능성이 높은 단일 출력 레코드가 나오고 <A,(321, null)>가 된다. 이 레코드는 집계의 내부 상태 저장소에 기록되고 다운 스트림 작업으로 전달된다.

캐시 크기는 아래와같이 설정 가능하다.
``` java
// Enable record cache of size 10 MB.
Properties streamsConfiguration = new Properties();
streamsConfiguration.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024L);
```
프로세서 토폴로지 인스턴스의 쓰레드가 캐시 크기를 나눠 갖는다. 쓰레드 간에 캐시 공유는 없다.

캐시를 얻으려면 put(), get() 할 수 있다. 캐시는 캐시 크기에 도달하면 LRU 방식으로 제거되어진다. 
캐싱의 의미는 데이터가 State Store 에 플러시 되고 commit.interval.ms 또는 cache.max.bytes.buffering 히트가 발생할 때 마다 다운 스트림 프로세서 노드로 전달된다는 것이다. 
![Alt text](https://monosnap.com/image/VJKg0wnVEwD6n5A8O3KuGomeStrn7y)

``` java
// 캐시를 끄려면 0으로 설정한다.
Properties streamsConfiguration = new Properties();
streamsConfiguration.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 0);
```
캐싱을 끄면 기본 RocksDB 저장소에 대한 쓰기 트래픽이 높아질 수 있다. 기본 설정을 사용하면 Kafka Streams 내에서 캐싱이 활성화되지만 RocksDB 캐싱은 비활성화된다. 따라서 높은 쓰기 트래픽을 피하려면 Kafka Streams 캐싱이 꺼져있는 경우 RocksDB 캐싱을 활성화하는 것이 좋다. 레코드의 캐싱 시간을 유지하려면 커밋 간격을 주면 된다.
``` java
Properties streamsConfiguration = new Properties();
// Enable record cache of size 10 MB.
streamsConfiguration.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024L);
// Set commit interval to 1 second.
streamsConfiguration.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
```

### Processor API 에서 레코드 캐시
Stateful 프로세서 노드에서 State Store 로 쓰여지기 전에 아웃풋 레코드의 Compacting 이 일어나고 인터널 캐싱이 일어난다. processor API 의 레코드 캐시는 하류 노드로 전달되는 아웃풋 레코드를 캐시하거나 압축하지 않는다. 모든 다운스트림 프로세서에서 모든 레코드를 볼 수 있다. 하지만 State Store 로 쓰여지기 전에는 캐싱되므로 State Store 에서는 줄어든 레코드를 볼 수 있다. State Store 에서 caching 을 하려면 아래와 같이 enable 을 해야한다. 기본 값은 disable 이다.
``` java
StoreBuilder countStoreBuilder =
  Stores.keyValueStoreBuilder(
    Stores.persistentKeyValueStore("Counts"),
    Serdes.String(),
    Serdes.Long())
  .withCachingEnabled()
```

### 다른 메모리 사용

* 프로듀서 버퍼링
* 컨슈머 버퍼링 fetch.max.bytes 또는 fetch.max.wait.ms 로 간접적으로 제어 가능.
* TCP 송수신 버퍼
* 비직렬화 객체 버퍼: consumer.poll() 로 얻은 레코드에서 타임스템프를 추출하기위해 버퍼링된다.
* on-heap, off-heap 에서 모두 사용되는 RocksDB 메모리.
	* 이 값은 block_cache_size, write_buffer_size와 max_write_buffer_number, rocksdb.config.setter 구성을 통해 설정 가능.

위에서 언급한적 있듯이 State Store 를 사용할 때 iterator 를 사용했다면 반드시 close 로 닫아야 메모리 리소스 해제가 된다.
그렇지 않으면 메모리 사용량이 OOM 에 도달할 때까지 증가한다.


아래는 운영시 알아야 할 것.
https://kafka.apache.org/11/documentation/streams/developer-guide/running-app
