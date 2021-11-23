---
layout:     post
title:      "Transactional id in Kafka"
comments: true
tags: [ kafka ]
---

## Kafka에서 트랜잭션이란?

Kafka는 0.11.0 버전부터 트랜잭션 기능을 제공했다. 트랜잭션은 스트림 처리를 위해 널리 쓰이고 있는 "read-process-write" 패턴의 exactly-once 처리를 보장하기 위해 만들어졌다.

"read-process-write" 패턴은 Kafka Consumer로 메시지를 수신(read)하고, 가공(process)하여 Kafka Producer로 메시지를 발송(write)하는 작업을 의미한다. 

트랜잭션 API는 아래의 상황에서 Exactly-once 처리를 보장한다.

1. 인스턴스가 A 메시지를 처리를 완료하기 전에 죽었다. 이후에 인스턴스가 다시 살아나면 A 메시지를 재처리할 수 있다. 트랜잭션 API는 이 상황을 막아주어 A 메시지가 한번만 처리될 수 있도록 한다.
2. 인스턴스 1이 A 메시지를 처리하다가 잠시 브로커와의 연결이 끊겼다. 그래서 리밸런싱되어 인스턴스 2가 A 메시지 처리를 시작한다. 인스턴스 1이 다시 연결되면 인스턴스 1과 2 모두 A 메시지를 처리하여 중복 처리가 발생할 수 있다. 트랜잭션 API는 이 상황을 막아주어 A 메시지가 한번만 처리될 수 있도록 한다. Kafka에서는 이 상황을 "zombie instances" 문제라고 한다.

주의할 점은 메시지를 가공(process)할 때 부수효과가 있으면 exactly-once 처리를 보장해주지 않는다는 것이다. 즉, Kafka는 Kafka가 제어할 수 있는 범위 내에서만 exactly-once를 보장한다. Kafka를 사용할 때 이 점을 항상 염두에 두어야만 한다.
>For instance, if the processing has side effects on other storage systems, the APIs covered here are not sufficient to guarantee exactly-once processing.

## Zombie fencing

"Zombie instances" 문제를 해결하려면 각 프로듀서의 `transactional.id`에 고유 식별자를 할당해야만 한다. Kafka는 프로세스가 재시작되더라도 이 id로 각 프로듀서 인스턴스를 식별할 수 있다.

어떠한 프로듀서 A가 `transactional.id`로 브로커에 등록되면 브로커는 `transactional.id`와 최신 epoch을 매핑해서 관리한다. 만약 새로운 프로듀서 B가 같은 `transactional.id`로 브로커에 새로 등록되면 epoch을 갱신한다. 브로커는 최신 epoch 값으로 등록한 B의 요청을 처리하고, A의 요청은 거절한다.

A를 Zombie라고 부르고, A의 요청을 거절하는 행위를 Zombie fencing이라고 부른다.

## 트랜잭션 메시지 수신 방식

`transactional.id`와 관련은 없지만 트랜잭션 메시지 수신 방식을 알아보자. 컨슈머의 `isolation.level`을 `read_committed`로 설정하면 컨슈머는 **트랜잭션이 아닌(non-transactional) 메시지**나 **커밋이 된 트랜잭션 메시지**만 수신한다. 

즉, 컨슈머에 메시지를 전달한 프로듀서가 어떤 타입인지에 따라 수신하는 방식이 다르다. 트랜잭션 프로듀서면 커밋이 된 메시지만 수신하여 exactly-once를 보장하고, 일반 프로듀서면 발송된 모든 메시지를 수신한다.

## `transactional.id` 지정 규칙

[alpakka-kafka](https://doc.akka.io/docs/alpakka-kafka/current/transactions.html) 문서를 보니 `transactional.id`는 어플리케이션의 각 인스턴스마다 유니크한 값을 지정해야 한다고 했다. 그러자니 여러가지 의문이 생겼다.

- 배포할 때마다 인스턴스별로 `transactional.id`에 UUID를 새로 지정하면 `transactional.id`를 지정하는 의미가 없어진다. 인스턴스가 N개라면 각 인스턴스가 fixed id를 가지고 있어야 하나?
- 물리 장비에 배포할 때 장비의 host를 id로 쓰면 가능할 것 같다. 그런데 Kubernetes 환경에서 배포를 하면 pod의 host가 매번 변경된다. 노드의 host를 쓰려면 모든 노드에 인스턴스를 1개씩 필수로 배포해야 한다. DaemonSet으로 배포하는 건 좀 이상한데?
- StatefulSet으로 배포하면 pod 단위로 fixed id를 가질 수 있다. Fixed id를 위해 StatefulSet을 쓰는 게 맞나?
- 마법을 부려서 Deployment 객체에서 fixed id를 사용할 수 있다고 해보자. 스케일 아웃을 하면 새로운 인스턴스가 생기고 리밸런싱이 될텐데 그러면 중복 처리가 발생하지 않나?

공식 문서를 보니 해답이 있었다. Kafka Streams는 static assignment를 위해 각 파티션에 전용 프로듀서 인스턴스를 사용하도록 한다. 그래서 각 인스턴스의 `transactional.id`에 파티션 번호를 넣어준다.
> The only way the static assignment requirement could be met is if each input partition uses a separate producer instance, which is in fact what Kafka Streams previously relied on.

[spring-kafka](https://docs.spring.io/spring-kafka/reference/html/#transactions) 공식 문서를 보니 같은 방식으로 구현하였다.
>  ..., unless the transaction is started by a listener container with a record-based listener. In that case, the transactional.id is `<transactionIdPrefix>.<group.id>.<topic>.<partition>`. 

스프링은 리밸런싱 상황도 알아서 처리해준다.
> when a rebalance occurs a new instance will use the same transactional.id and the old producer is fenced.
 
이 방식을 쓰면 진정한 exactly-once 보장을 할 수 있다. 단, 트랜잭션 보장을 위해 프로듀서 인스턴스를 불필요하게 많이 생성해야하는 단점이 있다. 이게 최선일까?
 
## KIP-447: Exactly-once 처리를 위한 프로듀서 확장성 개선

Kafka 2.5 버전부터 위의 단점이 개선되어 프로듀서 인스턴스를 많이 생성해야할 필요도 없고, `transactional.id`에 파티션 번호를 넣어줄 필요도 없어졌다. 동작 방식은 간단하다.

- 컨슈머에서 처음 `consumer.poll()` 메소드를 호출하면 offset을 가져오기 위해 group coordinator에 FetchOffset을 요청한다. 참고로 이 동작은 트랜잭션과 관련 없는 기본 동작이다.
- 컨슈머 A가 "read-process-write" 패턴으로 1번 파티션의 메시지를 처리하다가 컨슈머 B가 1번 파티션을 이어받았다고 가정해보자. 그리고 1번 파티션에는 아직 커밋되지 못한 pending 메시지가 남아있다.
- 컨슈머 B가 1번 파티션에 대해 `consumer.poll()`을 하면 FetchOffset을 요청한다.
- Transaction coordinator는 1번 파티션의 **pending 메시지가 모두 commit될 때까지 FetchOffset 요청을 거절한다.** 컨슈머 B는 FetchOffset 요청이 수락될 때까지 back-off와 retry를 반복한다.
- 만약 트랜잭션이 타임아웃되면 pending 메시지는 모두 제거된다. 이제 Transaction coordinator는 컨슈머 B의 FetchOffset 요청에 응답한다. 컨슈머 B는 `consumer.poll()`을 진행한다.

핵심은 Transaction coordinator가 latest stable offset 등의 컨슈머 그룹의 정보를 알도록 개선되었다는 것이다. Transaction coordinator는 더 이상 `transactional.id`에만 의존할 필요가 없다.

이 방식은 트랜잭션 타임아웃이 길어지면 처리 지연이 발생할 수 있다. 그래서 아래의 옵션 값을 조정해야한다.

- `transaction.timeout`을 10초로 권장한다. 디폴트는 1분인데 너무 길다. 이는 `session.timeout`(디폴트 10초)와 맞춘 값이다. (왜 맞춰야만 하는지는 이해 못함)
- 브로커의 `transaction.abort.timed.out.transaction.cleanup.interval.ms` 설정도 10초로 권장한다.

>Considering the default commit interval was set to only 100 milliseconds, we would doom to hit session timeout if we don't actively commit offsets during that tight window. So we suggest to shrink the transaction timeout to be the same default value as session timeout (10 seconds) on Kafka Streams, to reduce the potential performance loss for offset fetch delay when some instances accidentally crash.

참고로 [spring-kafka](https://docs.spring.io/spring-kafka/reference/html/#exactly-once)는 EOS mode를 제공하여 이전 방식과 개선된 방식을 모두 제공한다.

## 새로운 `transactional.id` 지정 규칙

그러면 `transactional.id`에 어떤 값을 넣어야 할까? [spring-kafka](https://docs.spring.io/spring-kafka/reference/html/#transaction-id-prefix) 문서에서는 각 인스턴스마다 고유한 값을 가지면 된다고 한다.
> With EOSMode.BETA it is no longer necessary to use the same transactional.id, even for consumer-initiated transactions; in fact, it must be unique on each instance the same as producer-initiated transactions.

매번 배포할 때마다 각 인스턴스에 UUID를 새로 할당하면 된다는 의미일까? 아니면 이전처럼 fixed id를 가지고 있으라는 의미일까? [어떤 글](https://medium.com/@milo.felipe/spring-boot-kafka-transactions-producerfencedexception-6595623d1bd1)을 보니 아예 `random.uuid`를 넣어버렸다.
> transaction-id-prefix: tx-${random.uuid}

이 글을 신뢰해도 될까? 트랜잭션 API가 Kafka Streams를 위해 만들어졌으니 공식 소스를 참고해보자. [소스](https://github.com/apache/kafka/blob/286126f9a53751285e6b71349cdb6abcba35ddff/streams/src/main/java/org/apache/kafka/streams/processor/internals/StreamsProducer.java#L118)를 보면 새로운 방식인`EXACTLY_ONCE_V2`에서 `processId`와 `threadId`를 썼다. `processId`가 무엇인지는 모르겠지만 `threadId`는 배포 후에 고정되는 id는 아닌 것 같다. `threadId`를 만드는 [코드](https://github.com/apache/kafka/blob/286126f9a53751285e6b71349cdb6abcba35ddff/streams/src/main/java/org/apache/kafka/streams/processor/internals/StreamThread.java#L329)를 찾아보니 역시 그래보인다. `threadId`를 넣은 [이유](https://github.com/apache/kafka/pull/8443/files#r405910636)도 적혀있다.

이전 버전인`EXACTLY_ONCE_ALPHA`에서는 `taskId`를 썼다. Task가 뭘까? [Kafka Streams](https://docs.confluent.io/platform/current/streams/architecture.html#parallelism-model) 문서를 보면 아래와 같이 나온다. Task는 n개의 파티션과 매핑되고, 그 매핑은 절대로 변경되지 않는다.

>Kafka Streams uses the concepts of stream partitions and stream tasks as logical units of its parallelism model.
>
>Each stream partition is a totally ordered sequence of data records and maps to a Kafka topic partition.
>
>The assignment of stream partitions to stream tasks never changes, hence the stream task is a fixed unit of parallelism of the application.

이제 정리를 해보자. Kafka Streams는 이전 버전에서 `transactional.id`를 파티션에 매핑하기 위해 `taskId`를 넣었다. `EXACTLY_ONCE_V2`부터는 그럴 필요가 없기 때문에 [프로듀서 생성 방식을 변경](https://github.com/apache/kafka/pull/8318)하고, `transactional.id`에서도 `taskId`를 뺐다.

즉, 각 인스턴스에 매번 UUID를 새로 할당하면 된다. 나중에 트랜잭션 API를 사용할 때 직접 검증을 해봐야겠다.

## Conclusion

- Kafka는 0.11.0 버전부터 트랜잭션 기능을 제공한다.
- 트랜잭션 API는 "read-process-write"패턴의 exactly-once 처리를 보장하기 위해 만들어졌다.
- **외부 저장소 사용 등의 부수효과가 있으면 exactly-once 처리를 보장하지 않는다.**
- 2.5 버전 미만에서는 `transactional.id`에 파티션 번호를 포함하고, 파티션마다 각각의 프로듀서를 생성해서 사용해야한다.
- 2.5 버전부터는`transactional.id`에 UUID를 넣어주면 된다.
- KIP(Kafka Improvement Proposals)는 설명이 정말 잘 되어 있다.
- **(주의) 필자는 트랜잭션을 운영에서 써본 경험이 없다.**

## References
- [Transactions in Apache Kafka](https://www.confluent.io/blog/transactions-apache-kafka/)
- [Improved Robustness and Usability of Exactly-Once Semantics in Apache Kafka](https://www.confluent.io/blog/simplified-robust-exactly-one-semantics-in-kafka-2-5/)
- [KIP-447](https://cwiki.apache.org/confluence/display/KAFKA/KIP-447%3A+Producer+scalability+for+exactly+once+semantics)
- [spriing-kafka](https://docs.spring.io/spring-kafka/reference/html/#transactions)
- <https://medium.com/@milo.felipe/spring-boot-kafka-transactions-producerfencedexception-6595623d1bd1>