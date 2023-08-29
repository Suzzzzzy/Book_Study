## CHAPTER 5: DESIGN CONSISTENT HASHING

```
논의내용)
안정 해시에 가상 노드 정도 적용해 본다면 데이터를 균등하게 나눠서 저장할 수 있을 것 같다는 생각입니다.
다이나모DB나 카산드라에 이런 기술이 적용되어 있다고 하는데, 써보신 분이 있을까요?
사실 써봤어도 이런 해시 알고리즘이 적용되서 관리한다 까지는 못 찾아 봤을 것 같습니다.
```

수평적 규모 확장성을 달성하기 위해서 요청 또는 데이터를 서버에 균등하게 나누는 것이 중요하다.
안정 해시는 이 목표를 달성하기 위해 보편적으로 사용하는 기술이다.

### 해시 키 재배치(rehash) 문제

N개의 캐시 서버가 있다면 이 서버에 부하를 균등하게 나누는 방법은 아래와 같은 해시 함수를 사용하는 것이다.

serverIndex = hash(key) % N

총 4대의 서버를 사용한다고 가정했을 때 각각의 키에 대해서 해시 값과 서버 인덱스를 계산하면 아래 표와 같다.

| key | hash | hash % 4 |
|-----|-------|-----------|
| key0 | 18358617 | 1 |
| key1 | 26143584 | 0 |
| key2 | 18131146 | 2 |
| key3 | 35863496 | 0 |
| key4 | 34085809 | 1 |
| key5 | 27581703 | 3 |
| key6 | 38164978 | 2 |
| key7 | 22530351 | 3 |

hask(key0) % 4 = 1 이면 클라이언트는 캐시에 보관된 데이터를 가져오기 위해 서버1에 접속해야 한다.
아래 그림은 키 값이 서버에 어떻게 분산되는지 보여준다.

<img width="416" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/56cb2c8c-05b4-44fa-8487-4b92fb22da5e">

이 방법은 서버 풀(server pool)의 크기가 고정되어 있고, 데이터 분포가 균등할 때는 잘 동작한다.
하지만 서버가 추가되거나 삭제되면 문제가 생긴다.
만약 서버 하나가 장애 때문에 중단 된다면 전체 서버 크기는 3이 되고 서버 인덱스 값은 달라지게 된다. 해시 %3의 결과이다.

| key | hash | hash % 3 |
|-----|-------|-----------|
| key0 | 18358617 | 0 |
| key1 | 26143584 | 0 |
| key2 | 18131146 | 1 |
| key3 | 35863496 | 2 |
| key4 | 34085809 | 1 |
| key5 | 27581703 | 0 |
| key6 | 38164978 | 1 |
| key7 | 22530351 | 0 |

변화된 키 분포는 아래 그림과 같다.

<img width="399" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/ab9493fb-acc5-455b-8911-cb58dcf2f624">

장애가 1번 서버에 일어나면 키가 재분배 된다. 캐시 클라이언트가 데이터가 없는 엉뚱한 서버에 접속하게 되고 대규모 캐시 미스가 일어나게 된다.
안정 해시는 이 문제를 효과적으로 해결하는 기술이다.

### 안정 해시

안정 해시(consistent hash)는 해시 테이블 크기가 조정될 때 평균적으로 오직 k/n개의 키만 재배치하는 해시 기술이다. k는 키의 개수이고, n은 슬롯의 개수다.
전통적인 해시 테이블은 슬롯의 수가 바뀌면 키를 재배치한다.

#### 해시 공간과 해시 링

동작 원리는 다음과 같다.
해시 함수로 SHA-1을 사용한다 가정하고 그 함수의 출력 값 범위는 x0, x1, ... xn이라 가정한다.
SHA-1의 해시 공간 범위는 0 부터 Pow(2, 160) - 1 까지이다.
따라서 x0는 0, xn은 Pow(2, 160) -1 이며 나머지는 그 사이 값을 갖게 된다.

이 해시 공간을 그림으로 표현하고 양쪽을 구부려 접으면 해시 링이 만들어진다.

<img width="451" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/38e3465b-63cb-4077-8e46-f70b8c1e35f4">
<img width="165" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/16cebedb-13ce-4c40-972b-53b8614d4aa1">

#### 해시 서버

해시 함수 f를 사용하면 서버 IP나 이름을 링 위의 어떤 위치에 대응시킬 수 있다.
4개의 서버를 해시 링 위에 배치하면 아래와 같다.

<img width="477" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/2a2c8695-6a3a-4b62-98eb-334538aae84f">

#### 해시 키

현재 해시 함수는 "해시 키 재배치 문제"에 언급한 함수와 다르며, 나머지 연산인 %를 사용하지 않고 있다.
아래 그림처럼 캐시 키 4개는 해시 링 위의 어느 지점에 배치할 수 있다.

<img width="462" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/94f90b7c-742e-442c-9ac0-2ba92bc5d227">

#### 서버 조회

어떤 키가 저장되는 서버는, 해당 키의 위치로부터 시계 방향으로 링을 탐색해 나가면서 만나는 천 번째 서버다.
아래 그림이 그 과정을 보여준다.

<img width="390" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/bbeaba34-f31f-4c04-b603-97f0be06f5e2">

#### 서버 추가

여기서 서버를 추가하더라도 키 가운데 일부만 재배치하면 된다.
아래 그림과 같이 서버4가 추가된 뒤에는 key0만 재배치된 상태를 보여준다.
이렇게 배치한 이유는 key0 위치에서 시계 방향으로 순회했을 때 처음으로 만나게 되는 서버가 서버4이기 때문이다.
다른 키들은 재배치되지 않는다.

<img width="471" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/f26d12b1-1543-47b7-bcda-9697babb27fb">

#### 서버 제거

아래 그림처럼 서버1이 삭제되면 key1이 서버2로 재배치됨을 알 수 있다.
시계 방향 순회에서 처음 만나는 서버가 서버2이며, 마찬가지로 다른 키들은 재배치되지 않는다.

<img width="462" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/23e12dff-b5f4-4d70-8fb1-c97ac86e83f9">

#### 기본 구현법의 두 가지 문제

안정 해시 알고리즘은 MIT에서 처음 제안되었다. 그 기본 절차는 다음과 같다.

- 서버와 키를 균등 분포(uniform distribution) 해시 함수를 사용해 해시 링에 배치한다.
- 키의 위치에서 링을 시계 방향으로 탐색하다 만다는 최초의 서버가 키가 저장될 서버다.

이 접근법에 두 가지 문제가 있는데,

첫째로 서버가 추가되거나 삭제되는 상황에 파티션의 크기를 균등하게 유지하는 게 불가능하다.
서버 별로 작은 해시 공간이나 굉장히 큰 해시 공간을 할당 받는 상황이 가능하다.
아래 그림은 s1이 삭제되서 s2의 파티션이 다른 파티션 대비 거의 두 배로 커지는 상황을 보여준다.

<img width="467" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/28f4d284-c5c1-4515-9af6-45427c260741">

두 번째로 키의 균등 분포를 달성하기가 어렵다.
아래 그림처럼 서버가 배치되어 있을 때, 서버1, 서버3은 아무 데이터도 갖지 않고 있고 대부분의 키는 서버2에 보관될 것이다.

<img width="462" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/6078c7b8-978b-4b64-9bbf-4cbc25d4f4c0">

이 문제를 해결하기 위해 제안된 기법이 가상 노드(virtual node) 또는 복제(replica)라 불리는 기법이다.

#### 가상 노드

가상 노드는 실제 노드 또는 서버를 가리키는 노드로, 하나의 서버는 링 위에 여러 개의 가상 노드를 가질 수 있다.
아래 그림 처럼 서버0과 서버1은 3개의 가상 노드를 갖는다.

<img width="465" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/bc47652d-7530-4c88-8dc9-c969c4063450">

3개는 임의의 수치이고 실제는 더 큰 값이 사용된다.
s0로 표시된 파티션은 서버이 관리하고,
s1로 표시된 파티션은 서버1이 관리하는 파티션이다.

키의 위치로 시계 방향으로 링을 탐색하다 만나는 최초의 가낭 노드가 해당 키가 저장될 서버가 된다.
아래 그림이 그 예이다. k0가 최초로 만나는 가상 노드는 s1_1이 나타내는 서버 즉, 서버1이 된다.

<img width="457" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/21d2c637-7807-4106-8917-84baf8560be0">

가상 노드 개수를 늘리면 키의 분포는 점점 더 균등해진다. 표준 편차(standard deviation)가 작아져서 데이터가 고르게 분포되기 때문이다. 표준 편차는 데이터가 어떻게 퍼져 나갔는지를 보이는 척도다.
가상 노드의 개수를 늘리면 표준 편차의 값이 떨어진다. 그러나 가상 노드 데이터를 저장할 공간은 더 많이 필요하게 된다.
타협적 결정(tradeoff)이 필요하다는 뜻이다. 그래서 시스템 요구사항에 맞는 가상 노드 개수를 적절히 조정해야 한다.

#### 재배치할 키 결정

다음과 같이 서버4가 추가 되었다고 하면 영향을 받는 범위는 s4 부터 s3까지이다.
즉, s3부터 s4 사이에 있는 키들을 s4로 재배치해야 한다.

<img width="464" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/f31e748d-2d40-42fc-8332-b0bf82d34b08">

서버 s1이 삭제되면 s1 부터 그 반시계 방향에 있는 서버 s0 사이에 있는 키들이 s2로 재배치되어야 한다.

<img width="462" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/86d6c8b1-d4d0-4179-a70a-dadaf5047dd4">

### 마치며

안정 해시의 이점은 다음과 같다.

- 서버 추가/삭제 시에 재배치되는 키의 수가 최소화된다.
- 데이터가 균등하게 분포하게 되므로 수평적 규모 확장성을 달성하기 쉽다.
- 핫스팟(hotspo) 키 문제를 줄인다. 유명 인사 문제와 같이 특정 샤드에 대한 접근이 빈번해 지면 과부하가 일어나므로 데이터를 좀 더 균등하개 분배해서 문제가 생길 가능성을 줄인다.

안정 해시는 실제로 널리 쓰이는 기술이고 몇 가지 예가 있다.

- Partitioning component of Amazon’s Dynamo database
- Data partitioning across the cluster in Apache Cassandra
- Discord chat application
- Akamai content delivery network
- Maglev network load balancer