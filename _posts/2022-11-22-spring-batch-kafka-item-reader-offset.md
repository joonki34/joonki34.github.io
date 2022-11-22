---
layout:     post
title:      "How KafkaItemReader in Spring Batch manages partition offsets?"
comments: true
tags: [ kafka, spring ]
---

## 1. Question

Spring Batch에서 아래와 같이 설정된 Job을 실행하고, 한번 더 재실행하니 이전에 처리한 메시지들을 그대로 다시 처리했다. 

```kotlin
return KafkaItemReaderBuilder<String, String>()
            .name("test")
            .topic(topic)
            .partitions(0, 1, 2)
            .consumerProperties(getConsumerProperties(groupId))
            .saveState(true)
            .build()
```
일반적인 Kafka 컨슈머라면 첫번째 실행 시점에 커밋된 offset부터 읽어야 하는데 왜 이럴까? `KafkaIteamReader`는 메시지를 수신하는 방식이 다른 것 같아서 찾아보기로 했다.

## 2. `auto.offset.reset` 문제인가?

회사 동료가 `auto.offset.reset`이 `earliest`로 잡혀있어서 그런 것 같다고 이야기해줘서 [문서](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#consumerconfigs_auto.offset.reset)를 찾아봤다. 

> What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server (e.g. because that data has been deleted):

위 문서를 보면 이 설정은 group id 기준 컨슈머의 offset이 브로커에 저장되어있지 않거나 저장된 현재 offset의 데이터가 브로커에 존재하지 않을 때 쓰여진다. `KafkaItemReader`는 offset을 커밋하고 종료하기 때문에 이 설정과는 무관하다. 모니터링 대시보드를 보니 lag도 없었다.

## 3. `KafkaItemReader` 동작 방식

Kafka 컨슈머 설정의 문제가 아니라면 `KafkaItemReader`의 문제일 것이다. 배치 콘솔 로그를 보니 `Seeking to offset 0 for partition`이라는 문구가 있었다. 배치가 실행되면 무조건 0부터 읽는 것 같다. 왜 이러는 걸까?

구글링을 해보니 비슷한 질문이 꽤 있었다. [Stack Overflow](https://stackoverflow.com/a/65882465) 답변을 보니 원래 이게 기본 동작인 듯하다. 커밋된 offset부터 읽을 수 있도록 Spring Batch 4.3부터 `setPartitionOffsets()`를 지원한다고 한다.

Spring Batch [Github 이슈](https://github.com/spring-projects/spring-batch/issues/3631)에서도 같은 문의를 볼 수 있었다. 그래서 `setPartitionOffsets()` 주석을 찾아봤다.

> Setter for partition offsets. This mapping tells the reader the offset to start reading from in each partition. This is optional, defaults to starting from offset 0 in each partition. Passing an empty map makes the reader start from the offset stored in Kafka for the consumer group ID.

> In case of a restart, offsets stored in the execution context will take precedence.

`partitionOffsets`를 지정하지 않으면 0부터 시작하고, 커밋된 offset부터 읽고 싶으면 empty map을 세팅해야 한다고 한다. 단, restart할 때는 execution context에 저장된 offset을 참조한다고 한다. 잡이 offset을 커밋할 때 execution context에도 저장한다.

공식 Repository [테스트 코드](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/test/java/org/springframework/batch/item/kafka/KafkaItemReaderTests.java#L250)에서도 `new HashMap<>()`를 넣어주고 있다.

그러면 이 설정된 `partitionOffsets`는 실제로 어떻게 쓰여질까? [코드](https://github.com/spring-projects/spring-batch/blob/c4ad90b9b191b94cb4d21f90e5759bda9857e341/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/kafka/KafkaItemReader.java#L169)를 찾아봤다.

```java
@Override
public void open(ExecutionContext executionContext) {
	this.kafkaConsumer = new KafkaConsumer<>(this.consumerProperties);
	if (this.partitionOffsets == null) {
		this.partitionOffsets = new HashMap<>();
		for (TopicPartition topicPartition : this.topicPartitions) {
			this.partitionOffsets.put(topicPartition, 0L); // (1)
		}
	}
	if (this.saveState && executionContext.containsKey(TOPIC_PARTITION_OFFSETS)) { // (2)
		Map<TopicPartition, Long> offsets = (Map<TopicPartition, Long>) executionContext
				.get(TOPIC_PARTITION_OFFSETS);
		for (Map.Entry<TopicPartition, Long> entry : offsets.entrySet()) {
			this.partitionOffsets.put(entry.getKey(), entry.getValue() == 0 ? 0 : entry.getValue() + 1);
		}
	}
	this.kafkaConsumer.assign(this.topicPartitions);
	this.partitionOffsets.forEach(this.kafkaConsumer::seek); // (3)
}
```

(1)을 보면 문서에 나온 것처럼 파티션별 offset을 0으로 세팅한다. [saveState](https://docs.spring.io/spring-batch/docs/current/reference/html/readersAndWriters.html#process-indicator)이 켜져있으면 재시작할 때 execution context에 저장된 값을 참조한다. (3)를 보면 `partitionOffsets`에 세팅된 정보로 `.seek()`을 실행한다. 즉, empty map으로 설정하면 `.seek()`을 실행하지 않아서 브로커에 커밋된 offset을 참조하게 된다.

## 4. Conclusion

매번 느끼지만 카프카 consumer 구현체들은 워낙 구현을 다양하게 해서 처리 방식을 잘 찾아보고 써야 한다. Spring Batch는 `KafkaItemReader` 공식 문서가 영 부실해서 정보를 찾기 쉽지 않았다.
