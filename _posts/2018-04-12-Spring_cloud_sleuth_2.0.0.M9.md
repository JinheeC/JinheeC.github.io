---
title: "Spring Cloud Sleuth (2.0.0.M9) - Zipkin 과 Brave"
categories:
- Zipkin
excerpt: |
  스팬 (Span) : 기본 작업 단위. 예를 들어 RPC(원격 프로시저 호출) 전송은 새로운 span 이다. 스팬은 스팬에 대한 고유 한 64 비트 ID와 스팬이 속한 추적에 대한 또 다른 64 비트 ID로 식별됩니다. Span에는 설명, 타임 스탬프 이벤트, 키 - 값 주석 (태그),  parent Spna ID 및 프로세스 ID (일반적으로 IP 주소)와 같은 다른 데이터를 가진다.
feature_text: |
  Zipkin 과 Brave에 대해 
feature_image: "https://picsum.photos/2560/600?random"
---

* table of contents
{:toc .toc}


### 전반적인 소개

스팬 (Span) : 기본 작업 단위. 예를 들어 RPC(원격 프로시저 호출) 전송은 새로운 span 이다. 스팬은 스팬에 대한 고유 한 64 비트 ID와 스팬이 속한 추적에 대한 또 다른 64 비트 ID로 식별됩니다. Span에는 설명, 타임 스탬프 이벤트, 키 - 값 주석 (태그),  parent Spna ID 및 프로세스 ID (일반적으로 IP 주소)와 같은 다른 데이터를 가진다.

스팬은 시작 및 중지 할 수 있으며 타이밍 정보를 추적한다. 스팬을 만든 후에는 반드시 나중에 스팬을 중지해야한다.
스팬을 시작하는 초기 스팬을 root span 이라고 한다. 이런 root span 의 ID값이 trace ID 가 된다.

Trace: span의 집합은 트리모양을 갖게 된다. 
annotation : 정확한 시간에 이벤트의 존재를 기록하는 데 사용되어지는데 Brave 를 사용하면 Zipkin 이 클라이언트와 서버의 대상, 요청을 시작한 위치 및 끝낸 위치를 이해하기 때문에 이를 위해 더 이상 특별한 설정할 필요가 없다. 그러나 어떻게 내부적으로 동작하는지 이해하기 위해 이러한 이벤트를 표시하여 어떤 종류의 작업이 수행되었는지 마킹하였다.

cs : Client Sent : 클라이언트가 보냄. 클라이언트가 요청함. 이 어노테이션은 범위의 시작을 나타냅니다.
sr : Server Received : 서버 쪽에서 요청을 받고 처리를 시작함. cs 로부터의 시간 차이가 네트워크 대기 시간을 보여준다.
ss : Server Sent. 요청 처리가 완료되면 주석이 달린다 (응답이 클라이언트로 되돌아온 경우). sr이 타임 스탬프에서 타임 스탬프를빼면서버 측에서 요청을 처리하는 데 필요한 시간이 나타난다.
![Alt text](https://monosnap.com/image/jy7Vpt4GplrlwxKKOHueRio83RKpmk)

같은 색의 의미는 같은 span 인데, 이 둘은 집킨 ui 에서 보면 알수 있듯이 머지되어진다. 어차피 이런 요청의 시작과 끝의 파악은 brave 가 해주니까 알필요는 없는데 그냥 내부에서는 그렇게 사용한다고...
![Alt text](https://monosnap.com/image/FTd6DtoFJ4oAb4VFskXSlWKaCGopFV)

2.0.0버전 부터는 spring cloud sleuth 가 내부적으로 트래이싱하기위해 brave를 사용한다. 그래서 슬루스는 더이상 문맥을 저장해 놓으려고 신경쓰지 않아도 되지만 그 일을 brave 한테 맞겨야 한다.

슬루스의 네이밍과 태깅 컨밴션이 brave 랑 다르기 떄문에 brave 의 컨밴션을 따르게 되었는데 만약 이전 버전의 슬루스와 같은 컨밴션을 사용하고 싶다면 spring.sleuth.http.legacy.enabled 를 true 로 설정해 주면 된다.

#### span context 의 전파(Propagating)
span 컨텍스트는 프로세스나 어플리케이션의 경계를 넘어서 모든 자식 스팬에 전파가 되어야 한다. 무조건 있어야 하는거는 Trace, spanID이고 그 이외의 것은 baggage(수하물) 이라고 부른다. 이 baggage 에 해당되는 값은 키 벨류 쌍으로 되어있고 프리픽스로 baggage- 로 시작한다.  이런 baggage 의 갯수 제한은 없지만 양이 커지면 대기시간이 길어진다는 건 알고 있어야 한다. 너무 많아지게 되면 어플리케이션 중단에 이를수도..

``` java
span initialSpan = this .tracer.nextSpan (). name ( "span" ) .start ();
try (Tracer.SpanInScope ws = this .tracer.withSpanInScope (initialSpan)) { 
	ExtraFieldPropagation.set ( "foo" , "bar" ); 
	ExtraFieldPropagation.set ( "UPPER_CASE" , "someValue" ); 
}
```

이런 baggage는 context 에 trace, spanid 와 함께 담겨서 차일드 스팬을 통과해서 돌아올때까지 이동하는데 zipkin 은 이런 baggage 에 대해 알지 못하므로 정보를 받지는 못한다. 

어플리케이션의 이름이 zipkin 에 찍히게 되는데 정확하게 찍히려면 bootstrap.yml에 spring.application.name 설정을 해주면 된다. 
> NOTE: span 마다 찍히는 로그는 아래와 같다. 순서대로 [appName, traceId, spanId, expotable] 에 해당한다.
> ![](https://monosnap.com/image/sRbYOXNa4Q0S6jqPntQHdlrMBexTYT)
> exportable: 로그를 zipkin 에게 보내야 하는지 여부. 

sleuth 는 http 또는 메세징 시스템의 경계를 트래이싱을 합치는 기본 로직이 있다. (예를들면 http 의 경우 헤더에 traceID 를 그대로 옮겨주고 parent를 만들어주는 등의 일을 말함.) 이런 전파 과정이 spanInjector, spanExtractor 처럼 디폴트로 정해져 있긴 하지만 커스터마이징도 할 수 있다. 

어노테이션을 이용하면 새로운 스팬을 만들거나 스팬을 이어지게 하거나 태그를 붙이는 등의 작업이 가능하다. 스팬의 이름은50자를 넘을 수 없다. 긴 이름은 지연의 원인이 될 수도 있고 예외를 던질수도 있으므로 주의.

zipkin 을 사용한다면 내보낼 확률을 설정해야 한다. spring.sleuth.sampler.probability (기본값 : 0.1, 즉 10 %). 


### Brave 소개
Brave는 분산 작업에 대한 대기 시간 정보를 캡처하여 Zipkin에보고하는 데 사용되는 라이브러리다. 대부분의 사용자는 Brave를 직접 사용하지 않는다. 대신 라이브러리 또는 프레임 워크를 사용한다.

이 모듈은 (잠재적으로) 분산 된 작업의 지연 시간을 모델링한 Span들을 만들고 조인하는 tracer를 가지고 있다. 네트워크 경계를 지나 trace 컨텍스트를 전파하는 라이브러리 (예 : HTTP 헤더 포함)도 포함한다.

우선 트래이싱을 하려면 Brave.Tracer 가 필요하다.


#### local 에서 추적 span 생성하기
새로운 root trace Id 를 가질 span일 경우 아래와 같이 작성한다.
``` java
Span span = tracer.newTrace().name("encode").start();
try {
  doSomethingExpensive();
} finally {
  span.finish();
}
```

새로운 trace가 아니라 child span이라면 아래와 같이 작성한다.
``` java
Span span = tracer.newChild(root.context()).name("encode").start();
try {
  doSomethingExpensive();
} finally {
  span.finish();
}
```

#### Span 커스터마이징 하기
Span 에는 태그를 달 수 있다. 이 태그들은 키 나 상세 정보를 넣을 수 도 있는데 아래 코드와 같이 작성이 가능하다.
``` java
span.tag("message.version", "1.2.3");
```
만약에 이 Span 을 유저(Span 의 시작 끝 등의 크리티컬한 작업을 하지 않을 유저)한테 건네줘야 하는 상황이 생긴다면 인터페이스 레벨로 건네주는 것이 좋은데 그 인터페이스 타입은 SpanCustomizer 인터페이스 이다. 

![text](https://steemitimages.com/400x0/https://monosnap.com/file/OZi9R58vLkig2pTlS1YuMiP40qgZ6h.png)

우리가 아는 Span 이라는 클래스는 SpanCustomizer 인터페이스를 구현한 추상 클래스이고 실제 데이터 타입은 AutoValue_RealSpan 이라는 클래스에 해당한다. 

> NOTE: ***Noop***과 Real 의 의미?
>  Noop 의 의미는 집킨 서버로 전송되는 실제 스팬이냐에 대한 여부에 해당한다. 실제 집킨으로 보내지는 스팬이라면 RealSpan일 것이고 그렇지 않다면 NoopSpan일 것이다.

아래는 SpanCustomizer의 메소드와 Span 의 메소드를 나열한 것이다. 메소드를 직접 보면 이해가 빠를 수 있다.

``` java
Span의 메소드
  public abstract boolean isNoop();
  public abstract TraceContext context();
  public abstract SpanCustomizer customizer();
  public abstract Span start();
  public abstract Span start(long timestamp);
  @Override 
  public abstract Span name(String name);
  public abstract Span kind(Kind kind);
  /** {@inheritDoc} */
  @Override 
  public abstract Span annotate(String value);
  /** {@inheritDoc} */
  @Override 
  public abstract Span annotate(long timestamp, String value);
  /** {@inheritDoc} */
  @Override 
  public abstract Span tag(String key, String value);
  public abstract Span remoteEndpoint(Endpoint endpoint);
  public abstract void finish();
  public abstract void abandon();
  public abstract void finish(long timestamp);
  public abstract void flush();


SpanCustomizer의 메소드
  SpanCustomizer name(String name);
  // 검색, 정보제공, 분석을 위해 태그를 달아놓을 수 있다. 예를 들면 버전정보를 넣는경우 버전별 스팬을 볼 수 있어서 유용하다. key 부분이 검색할 부분이고 value 는 null 일 수 없다.
  SpanCustomizer tag(String key, String value);
  // 이 스팬이 지연되는 시간에 대한 이벤트 설명을 적을 수 있다. 예를 들면 .annotate("process.retry")
  // 사용자는 집킨에서 이 어노테이션을 보고 이 스팬이 진행중인지 멈춘것인지 힌트를 받을 수 있다.
  SpanCustomizer annotate(String value);
  SpanCustomizer annotate(long timestamp, String value);
```

#### 원격 호출의 경우 호출 시점을 Span 에 추가하기
보통 원격 프로시저 호출은 인터셉터나 호출되는 시점을 지각하지 못할 때 비하인드 씬 처럼 호출되어지는 경우가 많다. 그래서 이 때 호출되는 시점에 대한 정보를 넣고 싶을 수도 있는데 이럴때는 아래와 같이 호출하는 클라이언트 쪽에서 스팬을 추가해 주면 된다.
``` java
span = tracer.newTrace().name("get").type(CLIENT);
span.tag("version", "6.36.0");
span.tag(TraceKeys.HTTP_PATH, "/api");  // 원하는 태그들을 설정하고
span.remoteEndpoint(Endpoint.builder()
    .serviceName("backend")
    .ipv4(127 << 24 | 1)
    .port(8080).build());

span.start();  // 스팬을 시작한다.

// 원격 프로시저 콜
// 응답이 돌아오면

span.finish();
```
위의 경우는 응답이 돌아오는 동기식 콜일 경우 가능한데 만약에 비동기의 콜의 경우에는 응답을 기다렸다가 finish()작업을 할 수 없다. 이 경우에는 대신에 flush() 메소드를 사용하면 된다.
``` java
AutoWired
Tracing tracing;
@AutoWired
Tracer tracer;

oneWaySend = tracer.newSpan(parent).kind(Span.Kind.CLIENT);

// Add the trace context to the request, so it can be propagated in-band
tracing.propagation().injector(Request::addHeader)
                     .inject(oneWaySend.context(), request);

// fire off the request asynchronously, totally dropping any response
request.execute();

// start the client side and flush instead of finish
oneWaySend.start().flush();
```
그리고 받는 쪽에서는 아래와 같이 사용이 가능하다.
``` java
// pull the context out of the incoming request
extractor = tracing.propagation().extractor(Request::getHeader);

// convert that context to a span which you can name and add tags to
oneWayReceive = nextSpan(tracer, extractor.extract(request))
    .name("process-request")
    .kind(SERVER)
    ... add tags etc.

// start the server side and flush instead of finish
oneWayReceive.start().flush();

// you should not modify this span anymore as it is complete. However,
// you can create children to represent follow-up work.
next = tracer.newSpan(oneWayReceive.context()).name("step2").start();

```
