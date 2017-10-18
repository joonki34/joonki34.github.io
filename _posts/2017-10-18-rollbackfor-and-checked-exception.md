---
layout:     post
title:      "@Transactional의 rollbackFor와 checked exception의 관계"
comments: true
tags: [ java ]
---

## Problem
`@Transactional`이 적용된 메소드에서 checked exception을 throw하도록 설정했는데 롤백이 되지 않았다.

## Solution
우선, `@Transactional`의 `rollbackFor` 옵션 문서를 찾아봤다. 문서에는 다음과 같이 적혀있었다.

기본적으로, 트랜잭션은 `RuntimeException`과 `Error`이 발생하면 롤백하지만 checked exception(비즈니스 로직 에러)에 대해서는 롤백하지 않는다. 자세한 설명은 `DefaultTransactionAttribute.rollbackOn(Throwable)`를 참고하라.
> By default, a transaction will be rolling back on RuntimeException and Error but not on checked exceptions (business exceptions). See DefaultTransactionAttribute.rollbackOn(Throwable) for a detailed explanation.

그래서 `DefaultTransactionAttribute.rollbackOn(Throwable)` 문서를 읽어봤다.

기본 동작은 EJB처럼 unchecked exception(`RuntimeException`)에 대해서는 비즈니스 로직에서 벗어난 결과라고 가정하고 롤백한다. `Error`도 예상하지 않은 결과이기 때문에 롤백을 시도한다. 반면에 checked exception은 트랜잭션 단위로 묶인 비즈니스 로직에서 예측 가능한 결과이다. (checked exception은 catch하여 대체 가능한 리턴값을 낼 수 있기 때문이다)
> The default behavior is as with EJB: rollback on unchecked exception (RuntimeException), assuming an unexpected outcome outside of any business rules. Additionally, we also attempt to rollback on Error which is clearly an unexpected outcome as well. By contrast, a checked exception is considered a business exception and therefore a regular expected outcome of the transactional business method, i.e. a kind of alternative return value which still allows for regular completion of resource operations.

이는 *선언되지 않은 checked exception*을 롤백한다는 점을 제외하면 `TransactionTemplate`과 동일하다. 선언형 트랜잭션에서 checked exception은 business exception을 의도적으로 선언한 것이라고 가정하고 롤백 없이 커밋한다.
> This is largely consistent with TransactionTemplate's default behavior, except that TransactionTemplate also rolls back on undeclared checked exceptions (a corner case). For declarative transactions, we expect checked exceptions to be intentionally declared as business exceptions, leading to a commit by default.

## Conclusion

Checked exception에 대해서 롤백을 수행하고 싶다면 `rollbackFor` 옵션에 해당 익셉션을 지정하면 된다. 하지만 그전에 이 익셉션이 checked exception의 정의에 부합하는지 먼저 고민해볼 필요가 있다. 

## Resources
<https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html#rollbackFor-->