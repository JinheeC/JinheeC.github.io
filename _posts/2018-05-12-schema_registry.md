---
title: "Confluent - Schema Registry 구축과정"
categories: 
- Kafka
excerpt: |
  환경에 맞는 방식대로 설치를 해준다. 아래는 압축 파일을 직접받아 설치한 방법이다.
  confluent-x.x.x 파일을 받았으면 해당 디렉토리로 이동해서 아래와 같이 실행하면 sample properties 를 가지고 뜨는 것을 확인 할 수 있다.
feature_text: |
  Confluent의 Schema Registry를 구축하는 과정에 대해
feature_image: "https://picsum.photos/2560/600?random"
---


* table of contents
{:toc .toc}



## 구축단계
로컬에서 설치하는 경우엔 1,2,3 과정은 필요하지 않다.

1. 개발 서버 발급
2. DNS 연결
3. 방화벽 오픈
4. A, B 섹션 순서대로 진행


## A. schema-registry 관련 작업
1. [confluent introduction](https://docs.confluent.io/current/schema-registry/docs/intro.html) 에서 환경에 맞는 방식대로 설치를 해준다. 아래는 압축 파일을 직접받아 설치한 방법이다.
2. confluent-x.x.x 파일을 받았으면 해당 디렉토리로 이동해서 아래와 같이 실행하면 sample properties 를 가지고 뜨는 것을 확인 할 수 있다.
3. `./bin/schema-registry-start ./etc/schema-registry/schema-registry.properties`
4. 이렇게 띄우면 되는데 원하는대로 설정을 바꾸려면 /etc/schema-registry/schema-registry.properties 를 복사해서 원하는 위치에 두고
5. 해당 파일을 수정하면 된다.
6. 아래 C 섹션에서 상세 프로퍼티에 대한 설명을 볼 수 있다.
7. 주요 설정 프로퍼티는 아래와 같다.

> 주요 **Properties**
> listeners=localhost:8081  // schema registry 주소를 말한다. confluent 로 설치한 registry 서버는 기본으로 8081 포트다.
>
> kafkastore.connection.url=localhost:2181   // kafka 의 zookeeper 주소다. 여러개가 있다면 콤마를 구분점으로 적어주면된다.
>
> kafkastore.topic=_schemas // schema 를 저장할 토픽 이름을 뜻한다 기본값이 _schemas 이다.
>
> debug=false  // api 호출이 실패할 경우 부가적인 디버깅 로그를 남기겠냐는 뜻이다.
>
> avro.compatibility.level=Full // 원하는 호환성을 넣어주면 된다. 기본은 backword. 개발 중이기 때문에 full 로 했다.
>
> access.control.allow.methods=GET,POST,PUT,DELETE,OPTIONS  // 이 값과 아래 값은 CORS 활성화를 위한 설정인데 자세한 내용은 아래 C 섹션에서 확인할 수 있다.
>
> access.control.allow.origin=*

8. 설정한 properties 3번 단계를 진행하면 된다.


## B. schema-registry-ui 관련 작업
1. github 에서 소스받아서 개발 서버에 받는다
2. 개발, 상용의 schema-registry clusters 를 명시하는 env.js 파일에 dev, prod(url 은 추후 수정 필요) 서버 정보 작성.
3. 위의 env.js 에 allowGlobalConfigChanges 설정, allowSchemaDeletion 설정을 할 수 있는데 아래와 같은 설정이다.
	* allowGlobalConfigChanges: global 호환성 레벨을 ui 에서 조정 가능하게 할 것인가
	* allowSchemaDeletion: schema 삭제를 ui에서 가능하게 할 것인가.
	
> allowGlobalConfigChanges 를 false 로 해놓으면 아래와 같이 ui 가 변경 불가능하게 되고 
>  ![Alt text](https://monosnap.com/image/GOxwC6Q7cBosNkev5DgRsOGLD9i7iJ)
> allowGlobalConfigChanges 를 true 로 해놓으면 ui 로 변경할 수 있게 된다.
> ![Alt text](https://monosnap.com/image/lJ7xlaZRq38xLKHE3NVpgNYo4TFbQ0)

4.   webpack.config.js 파일의 devServer 부분을 찾아 host 명을 맞게 변경한다.
5.  `npm install` 을 하여 node_modules 를 받고
6.  `npm run build-dev`를 한다.
7.  에러가 없으면 `npm run start` 를 해서 schema-registry-ui 를 띄운다.


## C. Schema Registry Properties

*  kafkastore.bootstrap.servers

Zookeeper서버를 명시하거나 또는 대안으로 Bootstrap server 즉 브로커 서버를 명시할 수 있다. 모든 조정 단계는 주키퍼 대신 브로커 서버를 통해서 진행된다. Master schema registry 인스턴스를 찾는데에도 쓰이고 등록된 스키마를 저장하는데에도 사용된다.
브로커 서버를 명시하는 방법은 zookeeper 서버를 명시하는 방법의 **대안** 일 뿐이다. 두 모드를 동시에 사용하지 않는것을 권장하고 있고. 이 모드는 새 배포에서만 사용하거나 모든 인스턴스를 종료하고 새 config로 전환 한 다음 스키마 레지스트리 인스턴스를 다시 시작할 때 사용하면 좋다.

*  listeners

http api 요청 수신을 위한 리스너. schema-registry 주소를 뜻한다. 

*  avro.compatibility.level

avro 호환성 설정 값을 뜻한다. 컨슈머 입장에서의 이름이라고 생각하면 된다.

    * none: 새 스키마는 다 유효함.
    * backword: 컨슈머가 v3 인데도 v1 으로 produce된 데이터를 읽을 수 있다는 뜻.
    * backward_transitive: 이전 버전 스키마에서 생선한 데이터를 다 읽을 수 있음. 컨슈머가 v3 면 v1, v2 로 produce 된 데이터를 다 읽는다.
    * forward: 가장 오래된 버전의 스키마에서 새 스키마를 읽는다. 컨슈머가 v1 인데도 새로운 스키마인 v3 로 produce된 데이터를 읽을 수 있다.
    * forward_transitive: 모든 이전에 등록된 스키마로 새 스키마 데이터를 읽을 수 있다. v3 로 produce 된 데이터를 v1, v2 인 컨슈머가 읽을 수 있다.
    * full: 앞뒤 호환. 뒤의 경우 가장 오래된 스키마를 읽을 수 있음.
    * full_transitive: 앞 뒤 호환가능. 오래된 스키마 전부 읽을 수 있음.


*  host.name

주키퍼가 알게 될 호스트 이름, 여러 노드의 스키마 레지스트리를 돌릴때만 필요함

*  kafkastore.ssl.key.password

키 저장소에 포함된 키 암호

*  kafkastore.ssl.keystore.location

ssl 키 파일 위치

*  kafkastore.ssl.keystore.password

키 스토어에 접근할 암호

*  kafkastore.topic

데이터 로그 역할을 하는 단일 파티션 토픽. 이 토픽은 retention policy 로 인한 데이터 유실을 피하기 위해 compated되어야 한다.

*  kafkastore.topic.replication.factor

스키마 토픽의 replication factor
기본값 : 3

*  ssl.keystore.location

https 에 사용될 ssl 키 저장소 파일 위치

*  ssl.keystore.password

https 에 사용될 키 스토어 파일 암호

*  ssl.key.password

https 에 사용될 키 암호

*  ssl.truststore.location

https 에 사용될 트러스트스토어 위치

*  ssl.truststore.password

https 에 사용될 트러스트스토어 암호

*  response.mediatype.preferred

가장 선호되는 순서로 응답에 사용되는 기본 미디어 타입
기본값 : [application / vnd.schemaregistry.v1 + json, application / vnd.schemaregistry + json, application / json]

*  zookeeper.set.acl

zookeeper ACL을 설정할지 여부

*  kafkastore.security.protocol

kafka 와 연결시 사용될 보안 프로토콜 기본 plaintext

*  kafkastore.ssl.enabled.protocols

SSL 연결에 사용할 프로토콜

*  kafkastore.ssl.keystore.type

키스토어 파일 형식

*  kafkastore.ssl.protocol

SSL 프로토콜을 사용하는지

*  kafkastore.ssl.provider

SSL에 사용되는 보안 공급자 어디인지

*  kafkastore.timeout.ms

Kafka 저장소에서의 작업 시간 초과

*  kafkastore.sasl.kerberos.service.name

Kafka 클라이언트가 실행되는 Kerberos 원칙 이름

*  kafkastore.sasl.mechanism

Kafka 연결에 사용되는 SASL 메커니즘

*  access.control.allow.methods

지정된 메소드에 대한 Access-Control-Allow-Origin 헤더 값 설정과 함께 명시해야 한다. get, post등 허용하고자하는 것들을 나열하면된다.

*  ssl.keystore.type

HTTPS에 사용되는 키 스토어 타입

*  ssl.truststore.type

HTTPS에 사용되는 트러스트 스토어 타입

*  ssl.protocol

HTTPS 사용

*  ssl.provider

HTTPS에 사용되는 SSL 보안 공급자

*  ssl.client.auth

HTTPS에 사용됨, 클라이언트가 서버의 트러스트 스토어를 통해 인증하도록 요구해야하는지 여부

*  ssl.enabled.protocols

SSL 연결에 사용할 수 있는 프로토콜 목록

*  access.control.allow.origin 

Access-Control-Allow-Origin 헤더 값 설정 CORS를 허용하는 것이다.
> CORS 란?
> Cross-Origin Resource Sharing 은 시스템 수준에서 타 도메인 간 자원 호출을 승인하거나 차단하는 것을 의미한다.

승인을 해야 다른 시스템에서의 호출이 가능하므로 허용해야한다.

*  debug

API 요청 오류 시 추가 디버깅 정보가 포함되는지 여부를 나타내는 부울

*  kafkastore.ssl.cipher.suites

SSL 암호 그룹 목록

*  kafkastore.ssl.endpoint.identification.algorithm

서버 인증서를 사용해서 서버 호스트 이름의 유효성 검증

*  kafkastore.ssl.keymanager.algorithm

key manager factory 가 SSL 접속에 사용하는 알고리즘

*  kafkastore.ssl.trustmanager.algorithm

트러스트 매니저 팩토리가 SSL 접속에 사용하는 알고리즘.

*  kafkastore.zk.session.timeout.ms

ZooKeeper 세션 시간 초과

*  metric.reporters

통계 리포터로 사용할 클래스 목록.

*  metrics.num.samples

메트릭을 계산하기 위해 유지 관리되는 샘플 수 
2

*  metrics.sample.window.ms

메트릭 시스템은 고정된 window 크기에 구성 가능한 샘플 수 를 유지하는데 예를 들면 30초의 경우 두개의 샘플이 유지된다. window가 만료되면 가장 오래된 window 가 지워지고 덮어쓴다.

*  port

DEPRECATED : 새 연결을 수신 대기하는 포트 대신 리스너를 사용해라

*  request.logger.name

NCSA 공통 로그 형식 요청 로그를 작성하는 SLF4J 로거의 이름.
기본:  "io.confluent.rest-utils.requests"

*  schema.registry.inter.instance.protocol

Schema Registry 인스턴스간에 호출하는 동안 사용되는 프로토콜. 마스터 노드에 대한 쓰기, 삭제요청이 이 프로토콜을 통해 되고 기본값은 http

*  schema.registry.resource.extension.class

SchemaRegistryResourceExtension 인터페이스의 유효한 구현의 완전 수식 클래스 이름이다. 필터와 같은 사용자 정의 리소스를 주입하는 데 사용할 수 있다. 일반적으로 로깅, 보안 등과 같은 사용자 지정 기능을 추가하는 데 사용된다.

*  schema.registry.zk.namespace

스키마 레지스트리 메타 데이터를 저장하기위한 ZooKeeper 네임 스페이스로 사용되는 문자열이다. 동일한 스키마 레지스트리 서비스의 일부인 스키마 레지스트리 인스턴스는 동일한 ZooKeeper 네임 스페이스를 가져야 한다.
기본값 : "schema_registry"

*  shutdown.graceful.ms

미해결 요청이 완료 될 때까지 종료 요청을 기다리는 시간
기본값:1000

*  ssl.keymanager.algorithm

key manaager factory가 ssl 접속에 사용하는 알고리즘

*  ssl.cipher.suites

SSL cipher suites

*  ssl.endpoint.identification.algorithm

엔드 포인트 식별 알고리즘은 서버 인증서를 사용하여 서버 호스트 이름의 유효성을 검증

*  kafkastore.sasl.kerberos.min.time.before.relogin

새로 고침 시도 사이의 로그인 시간 
기본 60000

*  kafkastore.sasl.kerberos.ticket.renew.jitter

갱신 시간에 추가 된 임의의 지터의 백분율.

*  kafkastore.sasl.kerberos.ticket.renew.window.factor

로그인 스레드는 마지막 새로 고침에서부터 티켓의 만료까지 지정된 윈도우 팩터에 도달 할 때까지 잠자기 상태가되고, 티켓 갱신을 시도


* kafkastore.group.id

이 설정을 사용하여 KafkaStore 사용자에 대한 group.id를 무시한다. 이 설정은 보안이 활성화되면 스키마 레지스트리 소비자의 group.id에 대한 안정성을 보장하는 데 중요해질 수 있다.
이 구성이 없으면 group.id는 "schema-registry- host -port"가 된다.



