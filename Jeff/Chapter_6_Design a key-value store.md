## CHAPTER 6: DESIGN A KEY-VALUE STORE

```
논의내용)
단순한 key-value 저장소 설계를 위해 많은 기술이 들어간다는 걸 (알고 싶지 않았지만) 알게 됐습니다.
뜬금없는 논란만 일어날 것 같지만, 자신이 개발하는 서비스가 일관성과 가용성 중에 어느 것이 더 중요하다고 생각하는지 얘기해 보면 좋겠습니다.
저는 실시간성을 요구하지 않는 서비스를 많이 개발하다 보니 가용성 보다는 일관성에 더 초점을 맞추고 개발을 했었습니다.
```

Key-value store는 키-값 데이터베이스라고 불리는 비 관계형(non-relational) 데이터베이스이다.
이 저장소에 저장되는 값은 고유 식별자(identifier)를 키로 가져야 한다.
키와 값 사이의 연결 관계를 "키-값" 쌍(pair)이라고 한다.

키-값 쌍에서의 키는 유일해야 하며 키에 해당하는 값은 키를 통해서만 접근할 수 있다.
키는 일반 텍스트 혹은 해시 값을 수 있지만 성능상의 이유로 짧을수록 좋다.

- 일반 텍스트 키: "last_logged_in_at"
- 해시 키: 253DDEC4

값은 문자열, 리스트, 객체일 수도 있다. 보통 값이 무엇인지는 상관이 없다.
널리 알려진 것으로 Amazon dynamo, Memcached, Redis가 있다.

표 6-1은 데이터 예시이다.

Table 6-1
| Key | value |
|-----|--------|
| 145 | john |
| 147 | bob |
| 160 | julia |

put(key, value)를 통해 키-값 쌍을 저장소에 저장하고
get(key)를 통해 키에 해당하는 값을 가져온다.

### 문제 이해 및 설계 범위 확정

There is no perfect design.
읽기, 쓰기, 메모리 사용량 사이에 균형을 찾고 데이터 일관성과 가용성 사이에 타협적 결정을 내린 설계를 만들었다면 쓸만한 답안일 것이다.
이번 예제에서는 다음 특성을 갖는 키-값 저장소를 설계해 본다.

- 키-값 쌍의 크기는 10KB 이하
- 큰 데이터 저장 가능
- 높은 가용성, 장애가 있어도 빠르게 응답해야 한다.
- 높은 확장성, 큰 데이터 셋에 따라 확장이 되어야 한다.
- 자동 확장성, 트래픽테 따라 자동으로 서버의 추가/삭제가 되어야 한다.
- 일관성 조정 가능
- 낮은 지연시간

### 단일 서버 키-값 저장소

서버 한 대로 설계하는 건 쉽다. 직관적으로 키-값 쌍 전부를 메모리에 해시 테이블로 저장한다.
속도는 빠르지만 모든 데이터를 메모리 안에 두는 것이 불가능하다는 약점도 있다.
이를 개선하기 위해서는 다음을 고려한다.

- 데이터 압축(compression)
- 자주 쓰이는 데이터만 메모리에 두고 나머지는 디스크에 저장

하지만 결국 서버 한 대로는 부족하다.
더 많은 데이터를 저장하기 위해서는 분산 키-값 저장소(distributed key-value store)가 필요하다.

### 분산 키-값 저장소

이것은 분산 해시 테이블이라고도 부른다. 키-값 쌍을 여러 서버에 분산시키기 때문에 그렇다.
분산 시스템을 설계할 때는 CAP 정리를 이해하고 있어야 한다.
CAP, Consitency Availability, Partition Tolerance theorem

#### CAP 정리

CAP 정리는 일관성, 가용성, 파티션 감내라는 세 가지 요구사항을 동시에 만족하는 분산 시스템을 설계하는 것은 불가능하다는 정리이다.
요구사항별 명확한 의미는 다음과 같다.

- Consistency: 클라이언트는 어떤 노드에 접속했는지 상관 없이 언제나 같은 데이터를 볼 수 있어야 한다.
- Availability: 클라이언트는 일부 노드에 장애가 발생해도 응답을 받을 수 있어야 한다.
- Partition Tolerance: 파티션은 두 노드 사이에 통신 장애가 발생했음을 의미한다. 네트워크에 파티션이 생기더라도 시스템은 계속 동작해야 한다.

아래 그림처럼 세 가지 요소 중에 두 가지를 충족하려면 나머지 하나는 반드시 희생되어야 한다는 것을 의미한다.

<img width="333" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/2c229503-1a1e-41f2-bcd6-0af663003382">

세 가지 중에 두 가지를 만족한다면 다음과 같은 분류가 가능하다.

- CP system: 일관성, 파티션 감내 지원, 가용성 희생
- AP system: 가용성과 파티션 감내 지원, 일관성 희생
- CA system: 일관성과 가용성을 지원, 파티션 감내 희생. 하지만 네트워크 장애는 피할 수 없으므로 분산 시스템은 어쩔 수 없이 파티션 문제를 감내해야 한다. 그러므로 실세계에서 CA 시스템은 존재하지 않는다.

이해를 위해 사례를 살펴본다.
세 대의 복제 노드 n1, n2, n3에 데이터를 복제하여 보관하는 상황을 가정한다.

<img width="349" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/c724744f-c057-486c-8c25-1319e815bb32">

##### 이상적 상태

네트워크가 파티션 되는 상황은 절대 일어나지 않는다. 따라서 n1에 기록된 데이터는 자동으로 n2, n3에 복제된다.
동시에 일관성, 가용성을 만족한다.

##### 실세계의 분산 시스템

분산 시스템은 파티션 문제를 피할 수 없다. 그리고 이 문제가 발생하면 일관성과 가용성 사이에 하나를 선택해야만 한다.
아래 그림은 n3에 장애가 발생해서 n1과 n2와 통신할 수 없는 상황이다.
n1, n2의 데이터는 n3에 전달 되지 않으며, n3에 있는 데이터 역시 n1, n2로 전달되지 않았다면 오래된 사본을 갖게 된다.

<img width="353" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/edd57da6-bbc0-4828-b0fe-67c7077dbaf4">

가용성 대신 일관성을 선택한다면(CP 시스템) 세 서버 사이에 생길 수 있는 데이터 불일치 문제를 피하기 위해 n1 n2에 쓰기 연산을 중단시켜야 하는데 그러면 가용성을 깨뜨리게 된다. 은행권 시스템은 데이터 일관성을 양보하지 않으므로 일관성이 깨질 수 있는 상황이 발생하면 상황을 해결하기 전까지 오류를 반환한다.

하지만 일관성 대신 가용성을 선택한 시스템(AP 시스템)은 오래된 데이터를 얻는다고 해도 계속 읽기 연산을 허용해야 한다. n1, n2에 계속 쓰기 연산을 허용하다가 n3의 문제가 해결된 뒤에 새 데이터가 n3에 전송될 것이다.

여태까지의 개념을 토대로 요구사항에 맞는 CAP 정리를 적용해야 한다.
면접때는 면접관과 상의하고 그 결론에 따라 시스템을 설계한다.

#### 시스템 컴포넌트

키-값 저장소 구현에 사용될 핵심 컴포넌트와 기술들을 알아본다.

- Data partition
- Data replication
- Consistency
- Inconsistency resolution
- Handling failures
- System architecture diagram
- Write path
- Read path

Dynamo, Cassandra, BigTable의 사례를 참고해서 다루는 내용들이다.

##### 데이터 파티션

데이터를 작은 파티션들로 분할한 다음에 여러 서버에 저장하는게 보통 해결책인데
데이터를 파티션 단위로 나눌 때 아래 두 가지 문제를 고려해야 한다.

- 데이터를 여러 서버에 고르게 분산할 수 있는가
- 노드가 추가되거나 삭제될 때 데이터의 이동을 최소화할 수 있는가

5장에서 안정 해시가 이런 문제를 푸는 데 적합한 기술이므로 동작 원리를 다시 살펴본다.

- 해시 링에 서버를 8대를 배치한다.
- 키-값 쌍을 어떤 서버에 저장할지 결정하기 위해 키를 같은 링 위에 배치한다. 링을 시계 방향으로 순회하다가 만나는 첫 번째 서버가 키-값 쌍을 저장할 서버로 선택한다. 따라서 key0는 s1에 저장된다.

<img width="415" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/4448b91e-0aec-4a95-96e7-f4613abf1fa2">

안정 해시를 사용해서 데이터를 파티션 하면 다음과 같은 장점이 있다.

- 규모 확장 자동화(automatic scaling): 시스템 부하에 따라 자동으로 서버 추가/삭제 되도록 만들 수 있다.
- 다양성(heterogeneity): 각 서버 용량에 맞게 가상 노드의 수를 조정할 수 있다. 고성능 서버는 더 많은 가상 노드를 갖도록 설정할 수 있다.

##### 데이터 다중화

높은 가용성과 안정성을 위해 데이터를 N개 서버에 비동기적으로 다중화(replication)할 필요가 있다. N은 튜닝 가능한 값이다.
아래 그림 처럼 N=3으로 설정한 배치에서 key0은 첫 N개의 서버에 데이터 사본을 저장하므로 s1, s2, s2에 데이터가 저장된다.

<img width="348" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/7d2a7bc4-82cc-45dc-82ae-3107ed3b9a1d">

가상 노드를 사용한다면 N개의 노드에 대응하는 실제 물리 서버의 개수가 N보다 작을 수 있다.
이 문제를 피하려면 같은 물리 서버를 중복 선택하지 않도록 해야 한다.

같은 데이터 센터에서 속한 노드는 정전, 네트워크 문제, 자연재해 등의 문제를 동시에 겪을 가능성이 있다.
그래서 데이터의 사본은 다른 센터의 서버에 보관하고, 센터들은 고속 네트워크로 연결한다.

##### 데이터 일관성

정족수 합의(Quorum Consensus) 프로토콜을 사용하면 읽기/쓰기 연산 모두에 일관성을 보장할 수 있다.

- N=사본 개수
- W=쓰기 연산에 대한 정족수, 쓰기 연산이 성공한 것이라면 W개의 서버로부터 쓰기 연산이 성공했다는 응답을 받아야 한다.
- R=읽기 연산에 대한 정족수, 읽기 연산이 성공한 것이라면 R개의 서버로부터 응답을 받아야 한다.

N=3인 경우에 대한 예제는 다음과 같다.

<img width="305" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/ffea4b09-ae39-456b-b048-2887ef8ecc35">

W=1은 데이터가 한 대의 서버에 기록된다는 뜻이 아니라 3개의 서버 중 최소 1개의 서버에서 쓰기 성공 응답을 받아야 한다는 뜻이다. 중재자(coordinator)는 클라이언트와 노드 사이에 프록시(proxy) 역할을 한다.
W, R, N의 값을 정하는 것은 응답 지연과 데이터 일관성 사이의 타협점을 찾는 전형적인 과정이다. W나 R이 1인 경우는 한 대의 서버의 응답만 받으면 되니까 응답 속도는 빠를 것이다. 1보다 크다면 데이터 일관성의 수준은 향상되지만 중재자는 가장 느린 응답 속도를 가진 서버를 기다려야 하므로 느릴 것이다.
W + R > N 인 경우에는 강한 일관성(strong consistency)이 보장된다. 일관성을 보증할 최신 데이터를 가진 노드가 최소 하나는 겹칠 것이기 때문이다.

면접 시에 N, W, R 값을 정하는데 있어서 몇 가지 구성을 참고할 수 있다.

- R=1, W=N: 빠른 읽기 연산에 최적화된 시스템
- W=1, R=N: 빠른 쓰기 연산에 최적화된 시스템
- W + R > N: 강한 일관성이 보장됨, 보통 N=3, W=R=2로 구성
- W + R <= N: 강한 일관성이 보장되지 않음

###### 일관성 모델

일관성 모델(consistency model)은 데이터 일관성의 수준을 결정하는데, 종류가 다양하다.

- 강한 일관성(strong consistency): 모든 읽기 연산은 가장 최근에 갱신된 결과를 반환한다. 클라이언트는 out-of-date 데이터를 볼 수 없다.
- 약한 일관성(weak consistency): 일기 연산은 가장 최근에 갱신된 결과를 반환하지 못할 수 있다.
- 최종 일관성(eventual consistency): 약한 일관성의 한 형태로, 갱신 결과가 결국 모든 사본에 반영 되는 모델.

강한 일관성은 모든 사본에 쓰기 연산의 결과가 완료될 때 까지 데이터 읽기/쓰기를 금지하는 것인데, 이렇게 하면 새로운 요청의 처리가 중단되므로 고가용성 시스템에는 적합하지 않다.
다이나모나 카산드라 같은 저장소는 최종 일관성 모델을 채택하고 있는데, 여기서 설계하는 예제 모델도 이 모델을 따른다. 이 모델에서 쓰기 연산이 병렬적으로 일어나면 저장된 값의 일관성이 깨질 수 있는데, 이 문제는 클라이언트가 해결해야 한다.

###### 비일관성 해소 기법: 데이터 버저닝

데이터를 다중화하면 가용성은 높아지지만 사본 간 일관성이 깨질 가능성은 높아진다. 버저닝(versioning)과 벡터 시계(vector clock)가 이 문제를 해소하는 기술이다.
버저닝은 데이터를 변경할 때 마다 해당 데이터의 새로운 버전을 만드는 것을 뜻한다. 따라서 각 버전의 데이터는 변경 불가능(immutable) 하다.
아래 그림과 같이 데이터의 사본이 노드 n1, n2에 보관되어 있는데, 이 데이터를 가져오려는 서버1, 서버2는 get("name") 연산의 결과로 같은 값을 얻는다.

<img width="381" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/c7b691d1-02ae-42f8-b01b-7345498b13bd">

그런데 다음과 같이 서버1이 "name"이 키인 값 "johnSanFrancisco"로 바꾸고, 서버2는 "johnNewYork"으로 바꾸는데, 이 연산이 동시에 이루어진다면 값이 충돌하게 된다. 충돌한 각 값을 버전 v1, v2로 할 수 있다.

<img width="387" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/71e24e45-310c-4dd7-8c94-f657e825ca66">

충돌이 일어나도 변경 이전의 값은 옛날 값이므로 무시할 수 있다. 하지만 v1, v2의 충돌은 해결하기 위해 버저닝 시스템이 필요하다. 여기서 벡터 시계가 이 문제를 푸는데 보편적으로 사용하는 기술이다.

벡터 시계는 [서버, 버전]의 순서쌍을 데이터에 연결 시킨 것이다. 어떤 버전이 먼저인지, 나중인지, 다른 버전과 충돌이 있는지 판별하는데 쓴다.
벡터 시계는 D([s1, v1], [s2, v2], ..., [sn, vn])과 같이 표현했을 때 D는 데이터, vi는 버전 카운터, si는 서버 번호이다.
데이터 D를 서버 si에 기록하면 시스템은 아래 작업 가운데 하나를 수행한다.

- [si, vi]가 있으면 vi 값 증가
- 없으면 새 항목 [si, 1]을 만든다.

이 로직이 수행되는 로직은 아래 그림과 사례를 통해 알아볼 수 있다.

<img width="390" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/7ec67001-0d78-4f16-be9f-449905137bb4">

- 클라이언트가 데이터 D1을 시스템에 기록, 이 쓰기 연산을 처리한 서버는 Sx이다. 벡터 시계는 D1([Sx, 1])로 변한다.
- 다른 클라이언트가 데이터 D1을 읽고 D2로 업데이트 한 다음 시스템에 기록, D2는 D1의 변경이므로 덮어쓴다. 서버는 같은 Sx가 처리한다면 벡터 시계는 D2([Sx, 2])로 바뀐다.
- 다른 클라이언트가 데이터 D2를 읽고 D3로 업데이트 한 다음 시스템에 기록, 이때 서버는 Sy가 처리한다. 그러면 벡터 시계는 D3([Sx, 2], [Sy, 1])로 바뀐다.
- 다른 클라이언트가 데이터 D2를 읽고 D4로 업데이트 한 다음 시스템에 기록, 서버는 Sz가 처리할 때, 벡터 시계는 D4([Sx, 2], [Sz, 1]일 것이다.
- 다른 클라이언트가 D3, D4를 읽으면 데이터 충돌이 있는 걸 알게 된다. D2를 Sy, Sz가 다른 값으로 바꿨기 때문이다. 이 충돌은 클라이언트가 해소한다. 이 연산을 처리한 서버가 Sx라면 벡터 시계는 D5([Sx, 3], [Sy, 1], [Sz, 1])로 바뀐다.

벡터 시계는 어떤 버전 X가 버전 Y의 이전 버전인지 쉽게 판단할 수 있다. 버전 Y에 포함된 값이 X에 포함된 값보다 같거나 큰지 보면 되기 때문이다. D([s0, 1], [s1, 1])은 D([s0, 1], [s1, 2])의 이전 버전이고 충돌은 없다.

어떤 버전 X와 Y 사이에 충돌은 Y의 벡터 시계 구성요소 가운데 X의 벡터 시계에서 동일 서버 구성 요소보다 작은 값을 갖는 것이 있는지 확인한다. D([s0, 1], [s1, 2])와 D([s0, 2], [s1, 1])은 서로 충돌한다.

벡터 시계에는 두 가지 분명한 단점이 있는데
첫 번째는 충돌 감지 및 해소 로직이 클라이언트에 들어가므로 구현이 복잡해진다.
두 번째는 [서버, 버전]의 순서쌍 개수가 굉장이 빨리 늘어난다. 길이에 임계치(threshold)를 설정하고, 임계치가 넘어가면 오래된 값을 제거하도록 해야 한다. 그런데 이렇게 하면 버전 간 선후 관계를 정확하게 결정할 수 없으므로 충돌 해소 과정의 효율성이 낮아진다. 다이나모를 쓰는 아마존은 실제 서비스에 그런 문제가 벌어지는 걸 발견한 적이 없다고 하므로 대부분 기업에서 벡터 시계는 적용해도 괜찮은 솔루션일 것이다.

###### 장애 처리

장애는 흔히 벌어지는 사건이므로 어떻게 처리할 것인지에 대한 문제는 매우 중요하다.
장애 감지(failure detection) 기법과, 장애 해소(failure resolution) 전략을 살펴본다.

###### 장애 감지

분산 시스템에서 한 대의 서버가 죽었다고 해서 바로 장애 처리를 하는 것이 아니라. 두 대 이상의 서버가 같이 서버가 죽었다는 보고가 있어야 장애가 있다고 간주한다.

아래 그림처럼 모든 노드 사이에 멀티캐스팀 채널을 구축하는 것이 장애를 감지하는 가장 손쉬운 방법이다. 하지만 서버가 많으면 많을 수록 비효율적이다.

<img width="311" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/cfa6ce73-8130-4266-9d29-060bbfb1c5ea">

가십 프로토콜(gossip protocol)은 분산형 장애 감지(decentralized failure detection) 솔루션으로 효율적인 관리가 가능하다.
동작 원리는 아래와 같다.

- 각 노드는 멤버십 목록(membership list)를 유지한다. 목록은 id와 박동 카운터(heartbeat counter) 쌍의 목록이다.
- 각 노드는 주기적으로 자신의 박동 카운터를 증가시킨다.
- 각 노드는 무작위로 선정된 노드들에게 주기적으로 자기 박동 카운터 목록을 보낸다.
- 박동 카운터 목록을 받은 노드는 멤버십 목록을 최신 값으로 갱신한다.
- 어떤 노드의 박동 카운터 값이 지정된 시간 동안 갱신되지 않으면 해당 노드는 장애 상태인 것으로 간주한다.

<img width="464" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/1fcfbcff-6744-462f-8ffa-3bee93681a48">

위 그림의 예제 설명이다.

- 노드 s0은 멤버십 목록을 가지고 있다.
- s0은 s2(id = 2)의 박동 카운터가 오랫동안 증가하지 않았다는 것을 발견한다.
- s0는 s2를 포함하는 박동 카운터 목록을 무작위로 선택한 다른 노드들에게 전달한다.
- 노드 s2의 박동 카운터가 오랫동안 증가되지 않았음을 발견한 모든 노드는 해당 노드를 장애 노드로 표시한다.

###### 일시적 장애 처리

장애가 감지되면 가용성을 보장하기 위해 조치를 해야 한다.
엄격한 정족수(strict quorum) 접근법이라면 읽기/쓰기 연산을 금지해야 한다.
느슨한 정족수(sloppy quorum) 접근법은 이 조건을 완화해서 가용성을 높인다. 정족수 요구사항을 강제하는 대신에 쓰기 연산을 수행할 W개의 서버와 읽기 연산을 수행할 R개의 서버를 해시 링에서 고른다. 물론 장애 서버는 무시한다.
장애 서버로 가는 요청은 다른 서버가 잠시 맡아서 처리한다. 그 동안 발생한 변경사항은 해당 서버가 복구 되었을 때 일괄 반영해서 데이터 일관성을 보존한다. 임시로 쓰기 연산을 처리한 서버에는 그에 관한 단서(hint)를 남겨둔다. 이런 장애 처리 방안을 단서 후 임시 위탁(hinted handoff) 기법이라고 한다.

아래 그림의 예제는 장애 노드s2에 대해 읽기/쓰기 연산은 일시적으로 s2가 처리한다. s2가 복구되면, s2는 생신된 데이터를 s2로 인계한다.

<img width="393" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/689cd427-ba66-4060-8ca2-6454a25d3ac3">

###### 영구 장애 처리

단서 후 임시 위탁 기법은 일시 장애 처리를 위한 기법이다.
영구 노드의 장애 상태의 처리는 반-엔트로피(anti-entropy) 프로토콜을 구현하여 사본들을 동기화한다.
반-엔트로피 프로토콜은 사본들을 비교하여 최신 버전으로 갱신하는 과정을 포함한다. 사본 간의 일관성이 망가진 상태를 탐지하고 전송 데이터의 양을 줄이기 위해서 머클(Merkle) 트리를 사용한다.

머클 트리는 해시 트리의 일종으로 각 노드에 그 자식 노드들에 보관된 값의 해시 또는 자식 노드들의 레이블로부터 계산된 해시 값을 레이블로 붙여두는 트리다. 해시 트리를 사용하면 대규모 자료 구조의 내용을 효과적이면서도 보안상 안전한 방법으로 검증할 수 있다.

키 공간이 1에서 12일 때 머클 트리를 만드는 예제를 한번 살펴본다.
일관성이 망가진 데이터가 위치한 상자는 다른 색으로 표시한다.

1단계: 아래 그림과 같이 키 공간을 4개의 버킷으로 나눈다.

<img width="454" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/b3c59fa2-2311-4f35-ad8f-0baf65c5e617">

2단계: 버킷에 포함된 각각의 키에 균등 분포 해시(uniform hash) 함수를 적용해서 해시 값을 계산한다.

<img width="466" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/845c19f8-83d6-48a9-bdc1-77fca14ab377">

3단계: 버킷 별로 해시 값을 계산한 후, 해당 해시 값을 레이블로 갖는 노드를 만든다.

<img width="460" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/7c144ebb-30ec-45ad-a902-8cb4c884cfcb">

4단계: 자식 노드의 레이블로부터 새로운 해시 값을 계산하고, 이진 트리를 상향식으로 구성해 나간다.

<img width="462" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/3f35030e-5b76-47d7-959e-4cc24c77434d">

이 두 머클 트리의 비교는 루트 노드의 해시값을 비교하는 것으로 시작한다.
해시 값이 일치하면 두 서버는 같은 데이터를 갖는 것이다.
값이 다르면 왼쪽 자식 노드의 해시값을 비교하고, 그 다음에 오른쪽 자식 노드의 해시 값을 비교한다.
결국 탐색하다 보면 다른 데이터를 갖는 버킷을 찾을 수 있으므로 그 버킷들만 동기화한다.

머클 트리를 사용하면 동기화해야 하는 데이터의 양은 실제 존재하는 차이의 크기에 비례하며, 두 서버에 보관된 데이터의 총량과는 무관하다. 하지만 실제 쓰이는 시스템의 경우 버킷 하나의 크기가 꽤 크다는 건 알아 두어야 한다.

###### 데이터 센터 장애 처리

데이터를 여러 데이터센터에 다중화한다.
한 데이터센터가 망가져도 다른 데이터 센터에 보관된 데이터를 이용한다.

##### 시스템 아키텍처 다이어그램

여태까지 기술적 고려사항들을 바탕으로 아키텍처 다이어그램을 그려본다.
아키텍처의 주된 기능은 다음과 같다.

<img width="461" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/363bb2d6-1821-4b25-a489-ead35e0537b7">

- 클라이언트는 키-값 저장소가 제공하는 get(key), put(key, value) API와 통신
- 중재자는 클라이언트에게 키-값 저장소에 대한 프락시 역할을 하는 노드
- 노드는 안정 해시의 해시 링 위에 분포
- 노드를 자동으로 추가/삭제 할 수 있게 시스템은 분산되어 있음(decentralized)
- 데이터는 여러 노드에 다중화한다.
- 모든 노드가 같은 책임을 지므로, SPOF(Single Point of Failure)는 존재하지 않는다.

분산 설계를 채택했으므로, 모든 노드는 아래 그림에 제시된 기능 전부를 지원해야 한다.

<img width="335" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/434886ab-2035-4313-bcb1-9d5a14f8b8f2">

##### 쓰기 경로

아래 그림은 쓰기 요청이 특정 노드에 전달되면 무슨 일이 일어나는지를 보여준다.
이 구조는 기본적으로 카산드라의 사례를 참고한 것이다.

<img width="467" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/5521bd61-93c8-4344-a58b-768a261bc5ea">

1. 쓰기 요청이 커밋 로그 파일에 기록된다.
2. 데이터가 메모리 캐시에 기록된다.
3. 메모리 캐시가 용량이 다 차거나 정해진 임계치에 도달하면 디스크에 있는 SSTable(Sorted-String Table)에 기록된다. 이것은 <키, 값.의 순서쌍을 정렬된 리스트 형태로 관리하는 테이블이다.

##### 읽기 경로

읽기 요청을 받은 노드는 데이터가 메모리 캐시에 있는지 살펴본다. 있다면 아래 그림과 같이 데이터를 클라이언트에 반환한다.

<img width="466" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/28652019-046d-49d2-8cab-0a27355282ec">

캐시에 없는 경우는 디스크에서 가져와야 한다. 어느 SSTable에 키가 있는지 알아내는 효율적인 방법이 필요한데 이런 문제를 푸는데 블룸 필터(Bloom filter)가 사용된다.
데이터가 메모리 캐시에 없다면 읽기 연산은 아래 그림과 같이 처리된다.

<img width="467" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/869ef3f8-f569-44e0-90d5-4d04b6cde9c8">

1. 데이터가 메모리에 있는지 검사, 없다면 2로 감
2. 블룸 필터를 검사
3. 블룸 플터를 통해 어떤 SSTable에 키가 보관되어 있는지 확인
4. SSTable에서 데이터를 가져옴
5. 해당 데이터를 클라이언트에 반환

### 요약

분산 키-값 저장소가 가져야 하는 기능과 기능 구현에 이용되는 기술을 정리해 본다.

| Goal / Problems | Technique |
|-------------------|------------|
| Ability to store big data | Use consistent hashing to spread the load across servers |
| High availability reads | Data replication Mult-Data center setup |
| Highly available writes | Versioning and conflict resolution with vector clocks |
| Dataset partition | Consistent Hashsing |
| Incremental scalability | Consistent Hashsing |
| Heterogeneity | Consistent Hashsing |
| Tunable consistency | Quorum consensus |
| Handing temporary failures | Sloppy quorum and hinted handoff |
| Handling permanent failures | Merkle tree |
| Handling data center outage | Cross-data center replication |