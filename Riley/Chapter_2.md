---
title: "가상 면접 사례로 배우는 대규모 시스템 설계 기초 Study [2장] 개략적인 규모 추정"
subtitle: ""
date: 2023-08-12T13:41:30+09:00
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

# 함께 논의하고 싶은 주제
---
- SNS 서비스인 우리 서비스는 DAU(Daily active users)를 산출하고 싶습니다. 우리 회사의 서비스의 활성유저는 어떻게 정의 할 수있을까요?


# 요약
---

## 2의 제곱수
볼륨의 단위를 2의 제곱수로 표현하면 어떻게 되는지 알아야 한다. 흔히 쓰이는 데이터 볼륨 단위

|2의제곱|근사치|이름|축약형|
|---	|---	|---	|---
|10|	1천만(thousand)|	1킬로바이트(Kilobyte)|	1KB
|20|	1백만(milion)|	1메가바이트(Megabyte)|	1MB
|30|	10억(bilion)|	1기가바이트(Gigabyte)|	1GB
|40|	1조(trilion)|	1테라바이트(Terabyte)|	1TB
|50|	1000조(quadrilion)|	1페타바이트(Petabyte)|	1PB

## 모든 프로그래머가 알아야 하는 응답 지연 값
이들 가운데 몇몇은 더 빠른 컴퓨터가 등장하면서 유효하지 않게 되었지만 아직도 이 수치들은 컴퓨터 연산들의 처리 속도가 어느 정도인지 짐작할 수 있도록 해준다.

- [Google Pro Tip: Use Back-Of-The-Envelope-Calculations To Choose The Best Design](http://highscalability.com/blog/2011/1/26/google-pro-tip-use-back-of-the-envelope-calculations-to-choo.html)

- L1 cache reference 0.5 ns
- Branch mispredict 5 ns
- L2 cache reference 7 ns
- Mutex lock/unlock 100 ns
- Main memory reference 100 ns
- Compress 1K bytes with Zippy 10,000 ns
- Send 2K bytes over 1 Gbps network 20,000 ns
- Read 1 MB sequentially from memory 250,000 ns
- Round trip within same datacenter 500,000 ns
- Disk seek 10,000,000 ns
- Read 1 MB sequentially from network 10,000,000 ns
- Read 1 MB sequentially from disk 30,000,000 ns
- Send packet CA->Netherlands->CA 150,000,000 ns 

### 주의사항
- 메모리는 빠르다 디스크는 아직도 느리다.
- 디스크 탐색`seek`은 피하라
- 단순한 압축 알고리즘은 빠르다
- 데이터를 인터넷으로 전송하기 전 압축하라
- 데이터 센터는 여러 지역에 분산되어 있고 센터들 간 데이터를 주고 받는데에 시간이 걸린다.
- 쓰기 연산은 읽기보다 40배 비싸다

## 가용성에 관계된 수치들
- 고가용성`high availability`은 시스템이 오랜 시간 동안 지속적으로 중단없이 운영될 수 있는 능력을 지청
- 퍼센트로 표현한다. (100%는 한번도 중단된 적이 없음을 의미한다.)
- 아마존, 구글, 마이크로소프트와 같은 사업자는 99% 이상의 SLA를 제공

### 트위터 QPS와 저장소 요구량 추청

#### 가정
- 월간 능동 사용자(Montly active user)는 3억(300million) 명이다.
- 50%의 사용자가 트위터를 매일 사용한다.
- 평균적으로 각 사용자는 매일 2건의 트윗을 올린다.
- 미디어를 포함하는 트윗은 10% 정도이다.
- 데이터는 5년간 보관한다.

#### 추정
QPS(Query per Second) 추정치
- DAU(3million) * 50% = 150 million
- QPS = 1.5 * 2 트윗/24시간/3600초 = 약 3500
- 최대 QPS(Peek QPS)=2 * QPS = 약 700

미디어 저장을 위한 저장소 요구량
- 평균 트윗 크니
  - tweet_id = 64Byte
  - 텍스트 = 140 byte
  - 미디어 = 1 MB

- 미디어 저장소 요구량 : 1.5억 * 2 * 10% * 1MB = 30TB/일
- 5년간 미디어로르 보관하기 위한 저장소 요구량 : 30TB * 365 * 5 = 55 TB

