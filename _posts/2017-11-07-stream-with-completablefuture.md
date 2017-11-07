---
layout:     post
title:      "Stream에서 CompletableFuture 사용하기"
comments: true
tags: [ java ]
---

## Problem

스트림 데이터를 각각 비동기로 별로의 쓰레드에서 병렬 처리하고, 결과를 취합하여 메인 쓰레드에 표시하는 작업을 구현해야했다. 그래서 다음과 같이 구현했었다.

```java
// 동물 단어를 받아서 각 단어들의 글자수의 합을 구한다고 가정해보자
Long sum = Stream.of("cat", "dog", "tiger", "lion")
                // calculate()은 단어의 글자수를 구하는 함수.
                // executor는 사전에 정의해둔 Executor. 쓰레드가 두개 이상인 Executor라고 가정하자.
                .mapToLong(word -> CompletableFuture.supplyAsync(() -> calculate(), executor).join())
                .sum();
System.out.println(sum);
```

하지만 막상 실행해보니 비동기적으로 처리되지 않았다. 곰곰이 생각해보니 저렇게 구현하면 스트림 데이터가 순차적으로 처리될테니 `CompletableFuture`를 쓴 의미가 없어진다. 그래서 다시 수정해보았다.

```java
// 동물 단어를 받아서 각 단어들의 글자수의 합을 구한다고 가정해보자
Long sum = Stream.of("cat", "dog", "tiger", "lion")
                // calculate()은 단어의 글자수를 구하는 함수.
                // executor는 사전에 정의해둔 Executor. 쓰레드가 두개 이상인 Executor라고 가정하자.
                .map(word -> CompletableFuture.supplyAsync(() -> calculate(), executor))
                .mapToLong(CompletableFuture::join)
                .sum();
System.out.println(sum);
```

이번에는 두 단계로 나눠서, 1단계에서는 `supplyAsync()`를 하고 그 후에 `join()`를 수행했다. 하지만 역시 안됐다. `map()`과 `mapToLong()` 모두 intermediate operation이기 때문이다. Terminal operation인 `sum()`에서 최종 처리될 것이므로 수정 전과 다를게 없다.

## Solution

이 [예제](http://fahdshariff.blogspot.kr/2016/06/java-8-completablefuture-vs-parallel.html)의 3, 4번을 참고하여 다시 수정해보았다.

```java
// 동물 단어를 받아서 각 단어들의 글자수의 합을 구한다고 가정해보자
List<CompletableFuture> futures = Stream.of("cat", "dog", "tiger", "lion")
                // calculate()은 단어의 글자수를 구하는 함수.
                // executor는 사전에 정의해둔 Executor. 쓰레드가 두개 이상인 Executor라고 가정하자.
                .map(word -> CompletableFuture.supplyAsync(() -> calculate(), executor))
                .collect(Collectors.toList()); // *중요*

Long sum = futures.stream()
                .mapToLong(CompletableFuture::join)
                .sum();

System.out.println(sum);
```

이렇게 하니 정상적으로 병렬 처리가 되었다. `collect()`로 스트림 자체를 terminate했기 때문에 `supplyAsync()`가 비동기 작업을 수행할 수 있었다. 그 후에 `join()`으로 취합하여 의도대로 동작하게 되었다.

## Thoughts

Stream의 intermediate/terminal operation에 대해서는 알고 있던 개념이었는데 `CompletableFuture`와 같이 쓰니 헷갈렸던 것 같다. `CompletableFuture`에 대한 이해가 부족하기도 하다. 그런데 생각보다 쓰기가 훨씬 복잡해서 좀 당황스럽다. 자바스크립트의 Promise와는 다르게 멀티쓰레드 방식이라 그런 것 같다. 이래서 Spring 5 WebFlux가 나왔나 싶기도..

## References
<http://fahdshariff.blogspot.kr/2016/06/java-8-completablefuture-vs-parallel.html>