## 2장 개략적인 규모 추정

```
논의 주제)
규모의 추정은 가정을 잘 하는 것에서 출발한다고 보고 싶습니다.
그랬을 때 트위터 예제에서 어떤 가정이 더 추가되면 좋을 것 같다고 생각하는지 얘기해 보면 좋을 것 같습니다.

저는 우선 지역별 규모를 더 추가하면 좋겠다는 생각이 있었습니다.
서비스를 하는 중심 지역이 어디고 그 지역의 사용자가 어느 정도인지에 따라서 서버 스케일을 다르게 가져가 볼 수 있다고 생각했고, 지역 사용자 별로 쓰는 데이터 형태도 다를 수 있다고 가정할 수 있다고 봤습니다. 예로 중국 사용자는 텍스트 형태 보다는 음성 메시지 형태를 더 선호하니까 음성 데이터 용량을 다른 지역 보다는 50%를 늘려 보는게 합리적인 가정이라고 생각할 수 있다고 봤습니다.
```

구글의 시니어 펠로(Senior Fellow) 제프 딘에 따르면

> "개략적인 규모 추정(back-of-the -envelope estimation)은 보편적으로 통용되는 성능 수치상에서 사고 실험(thought experiments)을 행하여 추정치를 계산하는 행위로서, 어떤 설계가 요구사항에 부합할 것인지 보기 위한 것" 이다.

개략적 규모 추정은 확장성을 표현하는 데 필요한 기본기에 능숙해야 한다.
특히, 2의 제곱수, 응답지연(latency) 값, 가용성에 관계된 수치들을 기본적으로 잘 이해하고 있어야 한다.

### 2의 제곱수

표 2-1은 데이터 볼륨 단위를 정리한 것이다.

Table 2-1
| Power | Approximate value | Full Name | Short Name |
|--------|----------------------|------------|---------------|
| 10 | 1 Thousnad | 1 Kilobyte | 1KB |
| 20 | 1 Million | 1 Megabyte | 1MB |
| 30 | 1 Billion | 1 Gigabyte | 1GB |
| 40 | 1 Trillion | 1 Terabyte | 1TB |
| 50 | 1 Quadrillion | 1 Petabyte | 1PB |

### 모든 프로그래머가 알아야 하는 응답지연 값

구글의 제프 딘은 2010년에 컴퓨터에서 구현된 연산들의 응답 지연 값을 공개한 적이 있다.
몇 가지 항목은 더 빠른 컴퓨터가 등장하면서 유효하지 않지만, 이 수치들은 연산 처리 속도가 어느 정도인지 짐작할 수 있게 해준다.

Table 2-2
| Operation name | Time |
|-------------------|-------|
| L1 cache reference | 0.5 ns |
| 분기 예측 오류(Branch mispredict) | 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 100 ns |
| Main memory reference | 100 ns |
| Compress 1K bytes with Zippy | 10,000 ns = 10 μs |
| Send 2K bytes over 1 Gbps network | 2,000 ns = 20 μs |
| Read 1 MB sequentially from memory | 250,000 ns = 250 μs |
| Round trip within the same datacenter | 500,000 ns = 500 μs |
| Disk seek | 10,000,000 ns = 10 ms |
| Read 1 MB sequentially from the network | 10,000,000 ns = 10 ms |
| Read 1 MB sequentially from disk | 30,000,000 ns = 30 ms |
| Send packet CA - Netherlands - CA | 150,000,000 ns = 150 ms |

이 수들을 알기 쉽게 시각화 해서 개발한 도구가 있다. 이건 2020년 기준으로 최근 기술 동향이 반영되어 있다.

<img width="456" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/ccb8874a-c4dc-4f03-8966-4e74788a0c71">

위 그림의 수치 분석 결론은 다음과 같다.

- Memory is fast but the disk is slow.
- Avoid disk seeks if possible.
- Simple compression algorithms are fast.
- Compress data before sending it over the internet if possible.
- Data centers are usually in different regions, and it takes time to send data between them

### 가용성에 관계된 수치들

고가용성(high availability)은 시스템이 오랜 시간 동안 지속적으로 중단 없이 운영될 수 있는 능력을 지칭하는 용어다.
%로 표현하고 100% 라면 한 번도 시스템이 중단된 적이 없다는 걸 의미한다. 대부분 99% ~ 100% 사이의 값을 갖는다.

SLA(Service Level Agreement)는 서비스 사업자가 사용하는 용어로, 고객들에게 합의한 수준을 말한다.
이 합의에는 공식적으로 가용시간(uptime)이 기술되어 있다. 대부분 클라우드 사업자는 99% 이상의 SLA를 제공한다.
관습적으로 9라는 숫자를 사용해 표시하며 9가 많을수록 좋다.
아래 표는 9의 개수와 시스템 장애 시간(downtime) 사이의 관계다.

Table 2-3
| Availability % | Downtime per day | downtiem per year |
|----------------|---------------------|-----------------------|
| 99% | 14.40 minutes | 3.65 days |
| 99.9% | 1.44 minutes | 8.77 hours |
| 99.99% | 8.64 seconds | 52.60 minutes |
| 99.999% | 864.00 milliseconds | 5.26 minutes |
| 99.9999% | 86.40 milliseconds | 31.56 seconds |

### 예제: 트위터 QPS와 저장소 요구량 추정

아래는 실제 트위터 성능이나 요구사항과는 상관이 없는 예제이다.

Assumptions:

- 300 million monthly active users.
- 50% of users use Twitter daily.
- Users post 2 tweets per day on average.
- 10% of tweets contain media.
- Data is stored for 5 years.

Estimations:

Query per second (QPS) estimate:

- Daily active users (DAU) = 300 million * 50% = 150 million
- Tweets QPS = 150 million * 2 tweets / 24 hour / 3600 seconds = ~3500
- Peek QPS = 2 * QPS = ~7000

We will only estimate media storage here.

- Average tweet size:
  - tweet_id 64 bytes
  - text 140 bytes
  - media 1 MB
- Media storage: 150 million * 2 * 10% * 1 MB = 30 TB per day
- 5-year media storage: 30 TB * 365 * 5 = ~55 PB

### 팁

개략적인 규모 추정과 관계된 면접에서 중요한 것은 올바른 절차로 문제를 풀어 나가는 것이다. 이건 결과를 내는 것 보다 중요하다. 답이 아니라 문제 해결 능력을 보여줘야 한다.
아래는 몇 가지 팁이다.

-  Rounding and Approximation: 복잡한 계산을 하는데 시간을 쓰는 건 낭비다. 계산 결과의 정확함이 아니라 근사치를 활용한다.
- 가정(assumtions)들은 나중에 살펴볼 수 있게 적어둔다.
- 단위(unit)를 붙여라. 5가 아닌 5KB 혹은 5MB로 확실히 적어야 나중에 헷갈리거나 모호함을 방지할 수 있다.
- QPS, 최대 QPS, 저장소 요구량, 캐시 요구량, 서버 수 등을 추정하는 문제가 많이 나온다. 미리 계산할 수 있게 연습 해 두자.