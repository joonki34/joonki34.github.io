---
layout:     post
title:      "InnoDB 엔진의 transaction과 autocommit 동작방식"
comments: true
tags: [ database, mysql ]
---

## Question

Hibernate에서 `@Transactional` 메소드 안에서 `saveAndFlush()`이 실행되는 동안 DB 관리툴(MySQL Workbench)에서 해당건을 `select` 쿼리를 실행해도 조회가 안됐다. `save()`로 바꿔도 마찬가지였다.

하지만 `@Transactional` 어노테이션을 떼면 둘 중 어느 메소드를 사용하든 조회가 된다. 그렇다면 트랜잭션 상에서는 `saveAndFlush()` 혹은 `flush()`로 실행된 쿼리는 트랜잭션이 끝날 때 데이터베이스에 반영이 되는 것인가? 그러면 `save()`와 다를게 없지 않은가?

## InnoDB의 쿼리 실행 방식

현재 사용하는 데이터베이스 엔진은 InnoDB이다. InnoDB에서는 모든 활동이 트랜잭션 안에서 발생한다. InnoDB에서는 autocommit 모드가 기본적으로 활성화되어있는데, 이 모드는 트랜잭션에 담기지 않은 쿼리를 자동으로 `commit`시켜주는 역할을 한다. 간단한 `select` 쿼리도 트랜잭션 안에서 실행된다는 뜻이다. 예를 들어, MySQL Workbench에서 `select * from Author;`를 실행하면 알아서 트랜잭션을 커밋해준다.

만약에 autocommit 모드를 끄면 어떻게 될까? MySQL Workbench에서 [다음](https://superuser.com/questions/317829/mysql-workbench-start-with-auto-commit-off)과 같은 방법을 이용해 autocommit 모드를 비활성화하고 `insert` 쿼리를 실행한 후에 다른 DB 관리툴(e.g. Sequel Pro)에서 조회해보면 조회가 안된다. 이유는 autocommit 모드를 껐기 때문에 아직 트랜잭션 커밋을 하지 않은 상태이기 때문이다. MySQL은 기본적으로 트랜잭션 레벨이 READ COMMITTED이기 때문에 다른 트랜잭션에서 추가한 컬럼이 보이지 않는다. `COMMIT;`으로 트랜잭션을 반영하면, 그 이후부터는 정상적으로 조회가 된다.

## 왜 `saveAndFlush()`가 반영이 안됐을까?

다시 질문으로 돌아오면, 트랜잭션이 끝나지 않은 상태에서 `saveAndFlush()`가 실행된 후에 관리툴에서 해당 건이 조회가 되지 않았다. 이 이유는 InnoDB 엔진 특성상 관리툴에서 `select` 쿼리를 실행하는 행위도 트랜잭션에 감싸서 실행되어지기 때문이다. 트랜잭션 레벨이 READ COMMITTED이기 때문에 커밋되지 않은 다른 트랜잭션의 데이터가 조회되지 않았던 것이다. 비록 조회는 안되지만 인덱스를 확인해보면 증가한 것을 알 수 있다. 결국 제대로 반영이 됐는데 안됐다고 착각한 것이었다.

## 왜 반영이 안된다고 생각했을까?

InnoDB의 특성을 몰랐던 탓이 가장 컸다. 또한, 무의식적으로 관리툴에서의 활동은 일종의 god mode라고 여기고 있었다. 관리툴에서 실행한 쿼리는 트랜잭션에 관계없이 실행된다고 생각했었다.

## References
<https://dev.mysql.com/doc/refman/5.6/en/innodb-autocommit-commit-rollback.html>