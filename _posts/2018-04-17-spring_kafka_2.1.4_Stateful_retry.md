---
title: "Spring Kafka 2.1.4 - Stateful Retry"
categories: 
- Kafka
excerpt: |
  상태가 있는 리트라이가 나오게 된 배경 부터 먼저 설명하자면, 지금 이 문서는 spring kafka 2.1.4 버전을 기준으로 작성하고 있는데 직전 버전인 2.1.3 에서 처음 나온 개념이다. 이 개념이 나온 이유는 `max.poll.interval.ms 초과 문제` 로부터 나오게 되었는데 이 문제는 아래와 같다.
feature_text: |
  Spring Kafka 2.1.4 의 Stateful Retry 에 대해서
---

* table of contents
{:toc .toc}

상태가 있는 리트라이가 나오게 된 배경 부터 먼저 설명하자면, 지금 이 문서는 spring kafka 2.1.4 버전을 기준으로 작성하고 있는데 직전 버전인 2.1.3 에서 처음 나온 개념이다. 이 개념이 나온 이유는 `max.poll.interval.ms 초과 문제` 로부터 나오게 되었는데 이 문제는 아래와 같다.

## max.poll.interval.ms 초과 문제
0.10.1.0 버전부터 스프링 카프카는 아래 두가지 정보로 컨슈머의 health 를 결정을 했다. 

* session.timeout.ms: 이 값으로 컨슈머가 Active 상태인가를 판단
* max.poll.interval.ms: 이 값으로 컨슈머가 Hang 걸려있는가를 판단 (기본 5분)

즉, 마지막 poll 이후에 5분동안 poll 이 일어나지 않으면 카프카는 이 컨슈머가 활동할 수 없다고 판단하고 다시 해당 파티션을 리발란싱 해서 다시 다른 컨슈머에게 assign 하게 된다. 그래서 retryTemplate 이 retryPolicy에 설정된 횟수만큼 재시도를 하는 중인데도 poll 이 없다는 이유로 re-assign이 될 수 있다는 것이다. 근데 문제는 retry 로직을 타고 있는 그 메세지가 commit 되지 않아서 위와 같은 과정으로 다른 컨슈머에게 re-assign 되었을 때 다시 해당 메시지를 건네서 retry 로직을 타게 되는 경우다. 이 경우 또 다시 시간초과로 다른 컨슈머에게 할당되어질 수 있다. 이런 문제가 있기 때문에 Stateful Retry 가 나오게 된 것이다. 이 문제는 특히나 리스너에서 api 호출로 다른 서버의 응답을 기다리는 로직이 있거나, RetryTemplate 에 BackOffPolicy 가 설정된 경우에 자주 발생 할 수 있다. BackOffPolicy 란 재시도 하기 전에 주어진 시간동안 기다리게 하는 정책이다. 주어진 시간이 지나면 재시도 -> 기다림 -> 재시도가 되는 것이다.

## Stateful Retry 란?
위의 문제를 이젠 SeekToCurrentErrorHandler 와 같이 Stateful Retry 로 피하면 된다. 
아래 코드와 같이 핸들러를 설정해주고 statefulRetry 를 true 로 설정하면 사용할 수 있고 이 경우 메시지를 retry 할 일이 생기면 에러를 발생시킨다. 이 에러를 받게되는 SeekToCurrentErrorHandler 가 어디까지 처리를 하고 재시도를 해야하는 상황인지 찾아서 그 처리되지 않은 오프셋(재시도할 오프셋) 부터 다음 poll() 에 넣어준다. 그리고 내부적으로는 재시도 정책(재시도 횟수 등...)에 맞추어서 해당 오프셋을 처리하게 한다. 
이렇게 poll()이 멈추지 않고 일어나기때문에 `max.poll.interval.ms 초과 문제` 는 일어나지 않는다.

``` java
Bean
public KafkaListenerContainerFactory<?> kafkaListenerContainerFactory() {
	ConcurrentKafkaListenerContainerFactory<Integer, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
	factory.setConsumerFactory(consumerFactory());
	factory.getContainerProperties().setErrorHandler(new SeekToCurrentErrorHandler() {

		@Override
		public void handle(Exception thrownException, List<ConsumerRecord<?, ?>> records,
			Consumer<?, ?> consumer, MessageListenerContainer container) {
			Config.this.seekPerformed = true;
			super.handle(thrownException, records, consumer, container);
		}

	});
	factory.setStatefulRetry(true);
	factory.setRetryTemplate(new RetryTemplate());
	factory.setRecoveryCallback(c -> {
		this.latch2.countDown();
		return null;
	});
	return factory;
}
```
