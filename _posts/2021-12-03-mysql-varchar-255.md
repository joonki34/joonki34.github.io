---
layout:     post
title:      "MySQL VARCHAR 255"
comments: true
tags: [ mysql ]
---

## Question

MySQL에서 인덱싱이 필요한 긴 문자열 컬럼을 정의할 때 관습처럼 `VARCHAR(255)`로 설정해왔다. 기존 테이블이 모두 255자로 정의되어있어서 그냥 그대로 썼다. 

그래서 문득 궁금증이 생겼다.

1. 256으로 하면 안되나? 왜 꼭 255였나?
2. `TEXT`도 인덱싱되지 않나? 왜 `VARCHAR`로 해야될까?

## 255, not 256.

MySQL 공식 문서를 보면 `CHAR`, `VARCHAR` 타입은 글자수를 저장하는 prefix와 데이터를 같이 저장한다. 그 중 `VARCHAR`는 글자수가 255면 1 바이트만 쓰고, 그 이상이면 2 바이트를 쓴다.

> VARCHAR values are stored as a 1-byte or 2-byte length prefix plus data. The length prefix indicates the number of bytes in the value. A column uses one length byte if values require no more than 255 bytes, two length bytes if values may require more than 255 bytes.

1 바이트는 2^8만큼의 숫자를 표현할 수 있다. 즉, 숫자 범위는 0 ~ 255이다. 참고로 `VARCHAR`는 최대 65,535까지 지정할 수 있다. 256^2 = 65,535이므로 아다리가 딱딱 맞는다.

## TEXT 타입 인덱스

그냥 긴 문자열을 `TEXT` 타입으로 지정하고 인덱스를 걸 수는 없을까? MySQL 공식 문서를 보면 `BLOB`이나 `TEXT` 타입 컬럼의 인덱스를 생성하려면 prefix 길이를 지정해야 한다.

```mysql
CREATE TABLE test (blob_col BLOB, INDEX(blob_col(10)));
```

그리고 최대 prefix 길이는 스토리지 엔진에 따라 다르다.

> Prefixes can be up to 767 bytes long for InnoDB tables that use the REDUNDANT or COMPACT row format. The prefix length limit is 3072 bytes for InnoDB tables that use the DYNAMIC or COMPRESSED row format. For MyISAM tables, the prefix length limit is 1000 bytes.

## References

- [The CHAR and VARCHAR Types in MySQL](https://dev.mysql.com/doc/refman/8.0/en/char.html)
- [The BLOB and TEXT Types in MySQL](https://dev.mysql.com/doc/refman/8.0/en/blob.html)
- <https://stackoverflow.com/questions/2340639/why-historically-do-people-use-255-not-256-for-database-field-magnitudes>
- <https://stackoverflow.com/questions/2889827/indexing-a-mysql-text-column>