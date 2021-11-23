---
layout:     post
title:      "리눅스 I/O 멀티플렉싱과 블록킹/논블록킹 디스크립터의 상관관계"
comments: true
tags: [ linux ]
---

I/O 멀티플레싱에서 select() 혹은 poll()는 파일 디스크립터가 readable/writable할 때까지 블록하고 사용가능한 상태가 되면 그때 return한다. 그래서 읽을 데이터가 있는 한 블록킹 디스크립터든 논블록킹 디스크립터든 read()에서 블록될 일이 없다.

file descriptor
블록킹 디스크립터에서 파일의 경우 파일 끝에 도달하면 EOF 값을 반환 받음. 하지만 소켓이나 디바이스 파일에서 읽은 경우에 읽기 시도(read)를 하면 파일처럼 끝 개념이 없으므로 다음 데이터가 들어올 때까지 블록됨. 그래서 논블록킹 쓰는 경우가 많음. 논블록킹은 EAGAIN 혹은 EWOULDBLOCK을 반환해서 다 읽은지 알 수 있음.

Level triggered 방식: 파일 디스크립터가 읽기 가능한 상태가 되었을 때마다 알림 받음
Edge triggered 방식: 읽을 수 있는 새로운 데이터가 들어올 때 알림 받음

select, poll, epoll(일부): level triggered 방식. 
epoll(일부): edge triggered 방식.

level triggered 방식
읽기 가능한 상태가 되었을 때마다 알림을 주기 때문에 블록킹 디스크립터을 쓸 경우 while문으로 read()를 할 필요가 없고 해서도 안됨 (블록될 수 있기 때문에). 그래서 select() 한번 돌 때 read() 한번 실행하는 식으로 하면 됨. 하지만 이렇게 하면 읽을 때마다 시스템 콜을 두번(select, read)하게 되므로 논블록킹 디스크립터를 사용해서 select() 한번 돌 때 반복문 돌려서 읽을 수 있을만큼 다 읽어서 시스템 콜을 줄이는게 좋음

edge triggered 방식
새로운 데이터가 들어올 때만 알림을 받기 때문에 level triggered처럼 매번 O(n)으로 디스크립터들을 검사하지 않아도 된다. 그래서 O(1)이다. 하지만 이 때문에 Edge triggered에서는 블록킹 디스크립터를 쓰면 오랜 기간 블로킹될 수 있다. 꼭 논블로킹으로 써야 함.

## References

<https://eklitzke.org/blocking-io-nonblocking-io-and-epoll>