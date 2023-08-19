---
title: "가상 면접 사례로 배우는 대규모 시스템 설계 기초 Study [7장] 분산 시스템을 위한 유일 ID 생성기 설계"
subtitle: ""
date: 2023-08-19T10:54:03+09:00
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

{{< admonition type=tip title="Note" open=true >}}
팀 내에서 진행하는 Study 정리 입니다.
{{< /admonition >}} 

# 함께 논의 하고 싶은 주제
---
- 다중 마스터 복제에서 언젠가는 값이 중복되지 않을까 하는데 어떻게 생각하시나요?
- 트위터 스노플레이크는 서버 갯수가 동적으로 줄었다가 늘었다가 하면 적용이 불가능한 걸까요?
- UUID는 신기하당


# 요약
---

## 1. 문제 이해 및 설계 범위 확정
- ID는 유일 해야 한다.
- ID는 숫자로만 구성되어야 한다.
- ID는 64비트로 표현
- ID는 발급 날짜에 따라 정렬 가능해야 한다.
- 초당 10,000 개의 ID를 만들 수 있어야 한다.

## 2. 개략적 설계안 제시 및 동의 구하기
분산 시스템에서 유일성이 보장되는 ID를 만드는 방법은 여러가지
- 다중 마스터 복제`multi-master replication`
- UUID`Universally Unique Identifier`
- 티켓 서버
- 트위터 스노플레이크 접근법

### 2.1 다중 마스터 복제
{{<figure src="/posts/images/system-design-interview/system-design-figure-7-2.png#center">}}
- 데이터베이스의 `auto_increment` 기능을 활용
- 값을 구할 때 1만큼 증가시켜 얻는 것이 아닌 k 만큼 증가 시킨다.

**단점**
- 여러 데이터 센터에 걸쳐 규모를 늘리기 어렵다.
- ID의 유일성은 보장되지만 그 값이 시간에 흐름에 맞추어 커지도록 보장할 수 없다.
- 서버를 추가하거나 삭제할 때도 잘 동작하도록 만들기 어렵다.

### 2.2 UUID
- 컴퓨터 시스템에 저장되는 정보를 유일하게 식별하기 위한 128비트 수다
- UUID의 충돌 가능성은 매우 낮다.
- `e9a07ed8-1a70-4328-b08f-a03e982a0baf`

**장점**
- UUID를 만드는 것은 단순하며 서버 사이 조율이 필요 없으므로 동기화 이슈도 없다
- 자기가 쓸 ID를 알아서 생성하는 구조로 규모 확장도 쉽다

**단점**
- ID 128비트로 길다 이번 장에서 다루는 문제의 요구사항은 64비트다
- ID를 시간순으로 정렬할 수 없다
- ID 숫자가 아닌 값이 포함될 수 있다.

### 2.2 티켓 서버
- `auto_increment` 기능을 갖춘 데이터베이스 서버, 즉 티켓서버를 중앙 집중형으로 하나만 참고 하는 것이다.

**장점**
- 유일성이 보장되는 숫자로만 ID 생성 가능
- 구현하기 쉽고 중소 애플리케이션에 적합하다
**단점**
- 티켓 서버가 SPOF가 된다.
- 이 이슈를 피하기 위해서는 티켓 서버 여러대를 준비해야 하는데 그렇게 되면 동기화 이슈가 발생

### 2.3 트위터 스노플레이크 접근법
- 사인`Sign` 비트 : 1비트를 할당. 지금으로서 쓰임새는 없지만 나중을 위해 유보
- 타임스탬프 비트 : 41비트를 할당. 기원 시각`epoch` 이후로 몇 밀리초가 경과했는지 나태는 값
- 데이터센터 ID : 5비트를 할당
- 서버 ID : 5비트를 할당
- 일련번호 : 12비트를 할당


## 4. 마무리

**시계 동기화`clock synchronization`**
- 우리는 ID 생성 서버가 모두 같은 시계를 사용한다고 가정
- NTP`Network Time Protocol` 

**각 절Select의 최적화**
- 동시성이 낮고 수명이 긴 애플리케이션이라면 일려번호 절의 길이를 줄이고 타임스탬프 절의 길이를 늘리는 것이 효과적


**고가용성`high availability`**
- ID 생성기는 필수 불가결 컴포넌트이므로 아주 높은 가용성을 제공


