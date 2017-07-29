---
layout:     post
title:      "트랜잭션 안에 있는 쿼리들은 각각 언제 실행될까?"
comments: true
tags: [ database, mysql ]
---

## Hypothesis
Spring Data JPA의 `saveAndFlush()`의 동작을 보던 중, 트랜잭션 안에 있는 쿼리들은 각각 언제 실행될지 의문이 생겼다. 내가 생각한 경우의 수는 두 가지였다. 

1. 모든 쿼리가 트랜잭션이 끝날 때 한꺼번에 실행된다
2. 쿼리가 각각 개별적으로 실행된다. 

## Experiment

그래서 MySQL에서 다음과 같은 쿼리를 실행해보기로 했다.
```sql
START TRANSACTION; # 트랜잭션 시작
INSERT INTO Author (id, name, gender) VALUES (999, 'David', 'Male'); # Author 테이블에 정보 입력
SELECT * from Author where id = 999; # id가 999인 row 선택
COMMIT; # 트랜잭션 커밋
```

만약 1번의 경우라면, id가 999인 row가 없을 것이므로 `SELECT` 절에서 에러가 나서 트랜잭션이 롤백될 것이다. 2번이라면 자연스럽게 row를 가져올 것이다.

## Conclusion

답은 2번이었다. 또한, 같은 조건에서 `COMMIT;`이 아닌 `ROLLBACK;`을 써서 강제로 롤백시킨 경우에는 auto increment 값이 올라간 것을 확인할 수 있었다. 롤백되기 전에 `INSERT` 쿼리를 실행했다는 뜻이다.

사실 모든 쿼리가 트랜잭션이 끝날 때 한꺼번에 실행된다면, Spring Data JPA에서 `saveAndFlush()`는 무의미할 것이고, Hibernate에서의 `flush()`도 무의미했을 것이다.