## CHAPTER 8: DESIGN A URL SHORTENER

```
논의내용)

논의 내용은 없고 생각에 대한 정리만 해봤습니다.

1. 단축 URL 설계 내용은 어렵지 않았지만 http status code 301과 302를 추가로 배운 것 같습니다.
2. 마무리에 데이터 분석 솔루션 내용을 보고 URL 단축기를 쓰는 이유를 생각해 볼 수 있었던 것 같습니다.
```

### 1단계 문제 이해 및 설계 범위 확정

면접관과 질문을 통해 모호함을 줄이고 요구사항을 알아낸다.

기본 기능

1. URL 단축: 주어진 긴 URL을 훨씬 짧게 줄인다
2. URL 리디렉션(redirection): 축약된 URL로 HTTP 요청이 오면 원래 URL로 안내
3. 높은 가용성과 규모 확장성, 그리고 장애 감내가 요구됨

#### 개략적 추정

- 쓰기 연산: 매일 1억개의 단축 URL 생성
- 초당 쓰기 연산: 1억 / 24 / 3600 = 1160
- 읽기 연산: 읽기 연산과 쓰기 연산 비율은 10:1, 그러면 읽기 연산은 초당 11600회 발생
- 10년간 운영한다고 가정 1억 x 365 x 10 = 3650억 개의 레코드를 보관해야 함
- 축약 전 URL의 평균 길이는 100
- 10년 동안 필요한 저장용량은 3650억 x 100 바이트 = 36.5TB

### 2단계 개략적 설계안 제시 및 동의 구하기

#### API 엔드포인트

엔드포인트는 REST 스타일로 설계한다.
URL 단축기의 엔드포인트는 두 개가 필요하다.

1. URL 단축용 엔드포인트: 단축 URL을 생성하려는 클라이언트는 단축 URL을 인자로 실어서 POST 요청을 보내야 한다.

POST /api/v1/data/shorten
인자: longUrl: longURLstring
반환: 단축 URL

2. URL 리디렉션용 엔드포인트: 단축 URL에 대해서 HTTP 요청이 오면 원래 URL로 보내주는 용도

GET /api/v1/shortUrl
반환: HTTP 리디렉션 목적지가 될 원래 URL

#### URL 리디렉션

아래 그림은 단축 URL을 입력하면 무슨 일이 일어나는지 보여준다.
단축 URL을 받은 서버는 그 URL을 원래 URL로 바꿔서 301 응답의 Location 헤더에 넣어 반환한다.

<img width="457" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/f98b879e-7148-4a9e-b77c-6b9ac246299d">

클라이언트와 서버 사이의 통신 절차에 대한 그림이다.

![image](https://github.com/jongfeel/BookReview/assets/17442457/771a8709-3f0b-4307-a06e-8de25bf8b31a)

여기서 301과 302의 응답의 차이는 다음과 같다.

- 301 Permanently Moved: 해당 URL에 대한 HTTP 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었다는 응답이다. 브라우저는 이 응답을 캐시한다. 이후 같은 단축 URL에 요청을 보내면 브라우저는 캐시된 원래 URL로 요청을 보낸다.
- 302 Found: 주어진 URL로 요청이 일시적으로 Location 헤더가 지정하는 URL에 의해 처리되어야 한다는 응답이다. 클라이언트의 요청은 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리디렉션되어야 한다.

첫 번째 요청만 단축 URL 서버로 전송하므로, 서버 부하를 줄이는 것이 중요하다면 301을 사용한다.
트래픽 분석이 중요할 때는 302를 쓰는 쪽이 클릭 발생률이나 발생 위치를 추적하는 데 좀 더 유리하다.

URL 리디렉션을 구현하는 직관적인 방법은 해시 테이블을 사용하는 것이다.

- 원래 URL=hashTable.get(단축URL)
- 301 또는 302 응답 Location 헤더에 원래 URL을 넣은 후 전송

#### URL 단축

긴 URL을 해시 값으로 대응시킬 해시 함수 fx를 찾는게 중요한 일이 된다.

<img width="236" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/589eeb47-5d0f-42d6-899d-d056d2db2771">

해시 함수는 다음 요구사항을 만족해야 한다.

- 입력으로 주어지는 긴 URL이 다른 값이면 해시 값도 달라야 한다.
- 계산된 해시 값은 원래 입력으로 주어졌던 긴 URL로 복원될 수 있어야 한다.

### 3단계 상세 설계

#### 데이터 모델

개략적 설계를 할 때는 해시 테이블을 사용했지만 실제 시스템에서 사용한다면 유한한 메모리 크기와 가격 때문에 곤란할 수 있다.
더 나은 방법은 < 단축 URL, 원래 URL >의 순서쌍을 관계형 데이터베이스에 저장하는 것이다.
실제 컬럼은 더 많을 수 있지만 중요한 것만 추려보면 id shortURL, longURL이 된다.

<img width="177" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/76b6ed78-204f-4199-a1d3-11e2d8360daf">

#### 해시 함수

해시 함수는 원래 URL을 단축 URL로 변환하는데 쓴다. 해시 함수가 계산하는 단축 URL의 값은 hashValue라 정해본다.

##### 해시 값 길이

hashValue는 [0-9, a-z, A-Z]의 무자들로 구성된다. 따라서 사용할 수 있는 문자의 개수는 10 + 26 + 26 = 62개다.
hashValue는 Pow(62, n) >= 3650억인 n의 최소값을 찾아야 한다.
아래 표는 hashValue의 길이와 해시 함수가 만들 수 있는 URL의 개수 사이의 관계를 나타낸다.

| n | Maximal number of URLs |
|---|-----------------------------|
| 1 | Pow(62, 1) = 62 |
| 2 | Pow(62, 2) = 3,844 |
| 3 | Pow(62, 3) = 238,328 |
| 4 | Pow(62, 4) = 14,776,336 |
| 5 | Pow(62, 5) = 916,132,832 |
| 6 | Pow(62, 6) = 56,800,235,584 |
| 7 | Pow(62, 7) = 3,521,614,606,208 = ~ 3.5 trillion |

n=7이면 3.5조 개의 URL을 만들 수 있다. 요구사항을 만족하므로 hashValue의 길이를 7ㄹ로 한다.
해시 함수 구현 기술로 '해시 후 충돌 해소', 'base-62 변환' 방법이 있다.

##### 해시 후 충돌 해소

쉬운 방법으로 CRC32, MD5, SHA-1과 같이 알려진 해시 함수를 이용한다.
아래 결과는 이 함수를 이용해서 "https://en.wikipedia.org/wiki/Systems_design"을 축약한 결과이다.

| Hash function | Hash value (Haxadecimal) |
|-----------------|-----------------------------|
| CRC32 | 5cb54054 |
| MD5 | 5a62509a84df9ee03fe1230b9df8b84e |
| SHA-1 | 0eeae7916c06853901d9ccbefbfcaf4de57ed85b |

CRC32는 7자리 보다 길기 때문에 계산된 해시 값에서 처음 7개 글자만 이용한다. 이렇게 하면 해시가 충돌할 확률이 높아지는데, 충돌이 발생하면 해소될 때 까지 사전에 정한 문자열을 해시 값에 덧붙인다. 이 프로세스는 아래 그림과 같다.

<img width="461" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/aa26cb22-05db-453f-b315-955de44f4d2e">

충돌 해소는 할 수 있지만 단축 URL을 생성할 때 한 번 이상 데이터베이스에 접근해야 하므로 오버헤드가 크다.
데이터베이스 대신 블룸 필터를 사용하면 성능을 높일 수 있다. 블룸 필터는 어떤 집합에 특정 원소가 있는지 검사할 수 있도록 하는, 확률론에 기초한 공간 효율이 좋은 기술이다.

##### base-62 변환

진법 변환(base conversion)은 URL 단축기를 구현할 때 흔히 사용되는 접근법 중에 하나이다. 이 기법은 수의 표현 방식이 다른 두 시스템이 같은 수를 공유하여야 하는 경우에 유용하다. 62진법을 쓰는 이유는 hashValue에 사용할 수 있는 문자 개수가 62개이기 때문이다.

10진수 11157을 62진수로 변환하는 절차는 다음과 같다.

- 62진법은 62개의 문자를 사용한다.
  - 0 ~ 9: 숫자 0 부터 9로 대응
  - 10 ~ 35: 영문자 a 부터 z로 대응
  - 36 ~ 61: 영문자 A 부터 Z로 대응
- 11157 = 2 x Pow(62, 2) + 55 x Pow(62, 1) + 59 x Pow(62, 0) = [ 2, 55, 59 ] => [ 2, T, X ]

![image](https://github.com/jongfeel/BookReview/assets/17442457/599a8054-2364-4442-a3e6-e85077fbe16d)

따라서 단축 URL은 https://tinyurl.com/2TX 가 된다.

##### 두 접근법 비교

아래 표는 두 접근법 사이의 차이를 요약한 내용이다.

| 해시 후 충돌 해소 전략 | base-62 변환 |
|--------------------------|-----------------|
| 단축 URL의 길이가 고정됨 | 단축 URL의 길이가 가변적, ID 값이 커지면 같이 길어짐 |
| 유일성이 보장되는 ID 생성기가 필요치 않음 | 유일성 보장 ID 생성기가 필요 |
| 충돌이 가능해서 해소 전략이 필요 | ID의 유일성이 보장된 후에야 적용 가능한 전략이라 충돌은 아예 불가능 |
| ID로부터 단축 URL을 계산하는 방식이 아니라서 다음에 쓸 수 있는 URL을 알아내는 것이 불가능 | ID가 1씩 증가하는 값이라고 가정하면 다음에 쓸 수 있는 단축 URL이 무엇인지 쉽게 알아낼 수 있어서 보안상 문제가 될 소지가 있음 |

#### URL 단축기 상세 설계

URL 단축키의 처리 흐름은 논리적으로 단순해야 하고 기능적으로 언제나 동작하는 상태로 유지되어야 한다.
base62를 써서 설계를 진행하고, 아래 그림과 같이 처리 흐름을 순서도로 표현한다.

<img width="402" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/f3dd46b0-88bb-44c3-8759-75ddf9ac735d">

1. 긴 URL 입력
2. 데이터베이스에서 해당 URL이 있는지 검사
3. 데이터베이스에 있다면 단축 URL이 있음 => 단축 URL을 클라이언트에 전달
4. 데이터베이스에 없다면 새 단축 URL을 위해 ID 생성 => 데이터베이스 기본 키
5. ID를 base 62 변환을 해서 단축 URL 생성
6. ID, 단축 URL, 원래 URL로 새 데이터베이스 레코드를 만든 후 단축 URL을 클라이언트에 전달

예제로 설명

- 원래 URL: https://en.wikipedia.org/wiki/Systems_design
- 이 URL에 대해 ID를 생성하면 2009215674938
- ID를 base62로 변환하면 zn9edcu
- 아래 표 처럼 데이터베이스에 추가

| ID | shortURL | longURL |
|----|-----------|-----------|
| 2009215674938 | zn9edcu | https://en.wikipedia.org/wiki/Systems_desing |

ID 생성기의 경우 단축 URL을 만들 때 사용하며 이 ID는 전역적 유일성(globally unique)이 보장되는 것이어야 한다.

#### URL 리디렉션 상세 설계

아래는 URL 리디렉션 매커니즘의 상세 설계 그림이다.
쓰기보다 읽기를 더 자주 하는 시스템이므로 < 단축 URL, 원래 URL > 의 쌍을 캐시에 저장하여 성능을 높인다.

<img width="460" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/f6021603-1f7d-4c43-915e-026badf453d5">

URL 리디렉션 흐름 요약

1. 사용자가 단축 URL 클릭
2. 로드밸런서가 웹 서버로 요청 전달
3. 단축 URL이 캐시에 있다면 원래 URL을 반환
4. 단축 URL이 캐시에 없다면 데이터베이스에서 가져온다. 데이터베이스에도 없으면 잘못된 단축 URL일 가능성이 있다.
5. 원래 URL을 캐시에 넣은 후에 사용자에게 반환

### 4단계 마무리

설계 이후 면접관과 더 고려해볼 사항

- 처리율 제한 장치(rate limiter): 단축 URL의 요청이 한번에 밀려올 경우 무력화될 수 있는 잠재적 보안 결함을 갖고 있다. 처리율 제한 장치를 통해 IP 주소를 비롯한 필커링 규칙을 이용해 요청을 걸러낼 수 있다.
- 웹 서버 규모의 확장: 무상태 계층으로 설계한 것이므로, 웹 서버의 추가/삭제가 자유롭다.
- 데이터베이스 규모 확장: 데이터베이스 다중화나 샤딩해서 규모 확장성을 달성할 수 있다.
- 데이터 분석 솔루션(analytics): URL 단축기에 데이터 분석 솔루션을 통합해 어떤 링크를 얼마나 많은 사용자가 클릭했는지, 언제 주로 클릭했는지 등 중요한 정보를 알아낼 수 있다.
- 가용성, 데이터 일관성, 안정성: 대규모 시스템이 잘 운영되기 위해서 갖춰야 하는 속성.