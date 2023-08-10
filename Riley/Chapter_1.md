---
title: "[1장] 사용자 수에 따른 규모 확장성"
subtitle: ""
date: 2023-08-10T09:43:42+09:00
author: "riley"
images: []
tags: ["System Design"]
categories: ["etc"]
fontawesome: true
linkToMarkdown: false
rssFullText: false
code:
  copy: true
  maxShownLines: 50
math:
  enable: false
share:
  enable: false
comment:
  enable: true
toc:
  auto: false
---

# 함께 논의하고 싶은 주제


# 요약

## 단일서버
모든 컴포넌트가 단 한대의 서버에서 실행되는 간단한 시스템

1. 도메인 이름을 이용하여 웹사이트에 접속. 도메인 이름은 도메인 이름 서비스(Domain Name Service, DNS)에 질의하여 IP 주소 반환 
2. DNS 질의 결과로 IP 반환
3. IP wnthfh HTTP(HyperText Transfer Protocol) 요청이 전달
4. 요청을 받은 웹 서버는 HTML 페이지 JSON 형태의 응답을 반환

## 데이터베이스

관계형 데이터베이스(Relational Data-base Management System, RDBMS)가 개발자들에게는 익숙하고 오랜기간 동안 잘 사용되어진 시스템이지만 구축하려는 시스템에 따라 꼭 최선의 시스템은 아닐 수 있다. 아래의 경우 비-관계형 데이터가 바람직한 선택이 될 수 있다.

비-관계형 데이터베이스가 적합한 서비스
- 아주 낮은 응답 지연시간`latency`이 요구
- 다루는 데이터가 비정형`unstructured`이라 관계형 데이터가 아님
- 데이터(JSON, YAML, XMAL 등)를 직렬화하거나`serialize` 역직렬화`deserialize` 할 수 있다
- 아주 많은 양의 데이터를 저장할 필요가 있다.

## 수직적 규모 확장과 수평적 규모 확장
- 수직적 규모 확장(`vertical scaling`) : `scale up`. 프로세서는 서버에 고사양 자원을 추가하는 행위
- 수평적 규모 확장(`scale out`) : `scale out`. 더 많은 서버를 추가하여 성능을 개선하는 행위

### 수직적 규모 확장의 단점
아래와 같은 단점때문에 대규모 애플리케이션을 지원하는 데는 수평적 규모 확장법이 적절하다.

- 확장의 한계가 있다. (CPU, 메모리를 무한대로 증설할 방법은 없다.)
- 장애의 자동복구(`failover`) 방안이나 다중화`re-dundancy` 방안을 제시하지 않음 
