---
layout:     post
title:      "Enable static analysis for Lombok's @NonNull annotation in IntelliJ"
comments: true
tags: [ tip ]
---

## Problem

IntelliJ에서 자바의 공식 `@Nonnull` 어노테이션에 대해서는 정적 분석을 지원하는데 Lombok의 `@NonNull` 어노테이션에 대해서는 지원이 안된다.

## Solution

[IntelliJ 문서](https://www.jetbrains.com/help/idea/2017.1/nullable-notnull-configuration-dialog.html)를 참고하여 `NotNull annotations` 항목에 Lombok의 `@NonNull` 어노테이션을 추가한다. 적용한 후부터는 해당 어노테이션에 대한 정적 분석이 지원된다.