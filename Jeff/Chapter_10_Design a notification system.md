## CHAPTER 10: DESIGN A NOTIFICATION SYSTEM

```
논의내용)
분산 환경에서 알림 시스템 설계는 제가 생각하지 못했던 부분을 알 수 있어서 좋았던 것 같습니다.
특히 한 가지 알림 방식이 아니라 제3자 서비스 여러개를 묶어서 할 수도 있다는 것도 알게 되었네요.

이번 챕터를 포함해서 여태까지 분석 서비스에 대한 실체를 언급하지 않고 있는데(심지어 다른 서비스들의 이름은 매우 잘 언급되어 있음!), 구글 애널리틱스 말고 이런 통합 서비스가 뭐가 있을까요?
```

이 기능을 갖춘 애플리케이션 프로그램은 최신 뉴스, 제품 업데이트, 이벤트, 선물 등 고객에게 중요할 만한 정보를 비동기적으로 제공한다. 일상생활의 중요한 부분으로 자리잡은 이 알림 시스템을 설계해 본다.
알린 시스템은 단순히 모바일 푸시 알림에 한정되지 않는다. 추가로 SMS, 이메일 까지 해서 세 가지로 분류할 수 있고 아래 그림으로 예제를 볼 수 있다.

<img width="439" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/a4ddcd0b-ccbf-4313-affc-ebeca363ca8e">

### 1단계 문제 이해 및 설계 범위 확정

알림 시스템이 어떻게 구현되는지는 깊은 이해가 필요하며, 정해진 정답이 없이 문제가 모호하게 주어지므로 적절한 질문을 통해 요구사항이 무엇인지 알아내야 한다.

면접관 답변 요약

- 푸시 알림, SMS 메시지, 이메일로 알림 지원
- 연성 실시간(soft real-time) 시스템으로 가정
- iOS, android, 랩톱/데스크톱 지원
- 알림 생성 주체: 클라이언트 애플리케이션 프로그램 혹은 서버 스케줄러
- 알림을 받지 않는 설정 필요
- 하루에 보낼 수 있는 알림 갯수: 모바일 푸시 1000만건, SMS 100만건, 메일 500만건

### 2단계 개략적 설계안 제시 및 동의 구하기

iOS, android, SMS, email 을 지원하는 알림 시스템의 개략적 설계안을 살펴본다.

#### 알림 유형별 지원 방안

##### iOS 푸시 알림

<img width="324" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/a9a5c1df-76a3-4573-86a9-69cfae0f37da">

iOS 푸시 알림은 세 가지 컴포넌트가 필요하다.

- 알림 제공자(provier): 알림 요청을 만들어 애플 푸시 알림 서비스(APNS, Apple Push Notification Service)로 보내는 주체
  - 단말 토큰(device token): 알림 요청을 보내는 데 필요한 고유 식별자
  - 페이로드(payload): 달림 내용을 담은 JSON dictionary

``` json
{
  "aps": {
                "alert": {
                   "title": "Game Request",
                   "body": "Bob wants to play chess",
                   "action-loc-key": "PLAY"
                },
                "badge": 5
             }
}
```

- APNS: 애플에서 제공하는 원격 서비스
- iOS 단말(iOS device): 푸시 알림을 수신하는 사용자 단말

##### 안드로이드 푸시 알림

안드로이드도 비슷한 절차로 진행된다. APNS 대신 FCM(Firebase CLoud Messaging)을 사용한다.

<img width="351" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/eb90a3b8-b24c-4dcf-b703-be56e6899be3">

##### SMS 메시지

트윌리오(Twilio), 넥스모(Nexmo) 같은 제3사업자의 서비스를 많이 이용한다. 상용 서비스이므로 이용요금이 부과된다.

<img width="358" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/8fd8cc43-016a-4118-b92b-57d7e9a1a71a">

##### 이메일

이메일 서버를 구축할 수 있지만, 상용 이베일 서비스를 이용한다. 샌드그리드(Sendgrid), 메일침프(Mailchimp)가 유명한 서비스이다. 전송 성공률도 높고, 데이터 분석 서비스도 제공한다.

<img width="348" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/d7da22f4-7b9d-4468-85f2-294933526b9e">

아래 그림은 지금까지 살펴본 알림 유형을 한 시스템으로 묶은 결과다.

<img width="239" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/07b8ef73-eedd-4b8c-8596-022cc6dbba39">

#### 연락처 정보 수집 절차

알림을 보내려면 모바일 단말 토큰, 전화번호, 이메일 주소 등의 정보가 필요하다.
아래 그림과 같이 사용자가 앱을 설치하거나 계정 등록을 하면 API 서버는 해당 사용자의 정보를 수집하여 데이터베이스에 저장한다.

<img width="463" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/9c8a13e2-664d-4bfc-9712-8a42b5f652d1">

연락처 정보 테이블 구조는 아래 그림과 같다. 필수 데이터만 담은 설계로 이메일 주소와 전화번호는 user 테이블, 단말 토큰은 device 테이블에 저장한다. 한 사용자는 여러 단말을 가질 수 있고, 알림은 모든 단말에 전송되어야 한다.

<img width="462" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/a0bc2fb8-9a20-4424-ae41-faf1b1789dbb">

#### 알림 전송 및 수신 절차

개략적으로 설계하고 점차 최적화해 나간다.

##### 개략적 설계안 (초안)

아래 그림은 개략적 설계 초안이다.

<img width="459" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/cbebc8f5-4e84-4437-8044-790e5b483995">

- 1 부터 N까지의 서비스: 서비스는 마이크로서비스, 크론잡, 분산 시스템 컴포넌트일 수도 있다. 납기일, 배송 알림 서비스가 예이다.
- 알림 시스템: 알림 시스템은 알림 전송/수신 처리의 핵심이다. 서버를 1개만 쓴다면 1 ~ N개의 서비스에 알림 전송을 위한 API를 제공해야 하고, 제3자 서비스에 전달할 알림 페이로드를 만들어 낼 수 있어야 한다.
- 제3자 서비스: 사용자에게 알림을 실제로 전달하는 역할. 통합 진행할 때 유의할 점은 화장성(extensibility)이다. 쉽게 새로운 서비스를 통합하거나 기존 서비스를 제거할 수 있어야 한다. 또 어떤 서비스는 다른 시장에서 사용할 수 없는데 FCM 같은 경우는 중국에서 사용할 수 없다. 중국에서는 JPush, PushY 같은 서비스를 사용해야 한다.
- iOS, android, SMS, email 단말: 사용자는 자기 단말에서 알림을 수신한다.

이 설계에는 몇 가지 문제가 있다.

- SPOF(SIngle-Point-Of-Failure): 알림 서비스에 서버가 하나밖에 없다면 장애가 생길 때 전체 서비스의 장애로 이어진다.
- 규모 확장성: 한 대의 서비스로 푸시 알림에 관계된 모든 걸 처리하므로, 데이터베이스나 캐시 등 중요 컴포넌트의 규모를 개별적으로 늘릴 방법이 없다.
- 성능 병목: 알림을 처리하고 보내는 것은 자원을 많이 필요로 하는 작업일 수 있다. 서버가 한 대인데 사용자 트래픽이 많이 몰리는 시간에는 시스템이 과부하 상태에 빠질 수 있다.

##### 개략적 설계안 (개선된 버전)

앞선 문제를 다음과 같은 방향으로 개선해 본다.

- 데이터베이스와 캐시를 알림 시스템의 주 서버에서 분리한다.
- 알림 서버를 증설하고 자동으로 수평적 규모 확장이 이루어질 수 있도록 한다.
- 메시지 큐를 이용해 시스템 컴포넌트 사이의 강한 결합을 끊는다.

아래 그림은 이 아이디어를 적용할 시스템 개선안이다.

<img width="440" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/ff01a8a7-26e0-4470-a100-b1275f9f7991">

- 1부터 N까지의 서비스: 알림 시스템 서버의 API를 통해 알림을 보낸 서비스
- 알림 서버(notification server)
  - 알림 전송 API: 스팸 방지를 위해 사내 서비스 또는 인증된 클라이언트만 이용 가능하다.
  - 알림 검증: 이메일 주소, 전화번호 등에 대한 기본 검증
  - 데이터베이스 또는 캐시 질의: 알림에 포함시킬 데이터를 가져오는 기능
  - 알림 전송: 알림 데이터를 메시지 큐에 넣는다. 하나 이상의 메시지큐를 사용하므로 알림을 병렬적으로 처리할 수 있다.

이메일 알림을 보내는 데 사용하는 API 예제는 다음과 같다.

POST https://api.example.com/v/sms/send

Request body:

``` json
{
  "to": [
    {
      "user_id": 123456
    }
  ],
  "from": {
    "email": "from_address@example.com"
  },
  "subject": "Hello, World!",
  "content": [
    {
      "type": "text/plain",
      "value": "Hello, World!"
    }
  ]
}
```

- 캐시: 사용자 정보, 단말 정보, 알림 템플릿 등을 캐시한다.
- 데이터베이스: 사용자, 알림, 설정 등 다양한 정보를 저장
- 메시지 큐: 시스템 컴포넌트 간 의존성을 제거하기 위해 사용. 다량의 알림이 전송되어야 하는 경우를 대비한 버퍼 역할도 한다. 종류 별로 메시지 큐를 사용했으므로 하나의 서비스에 장애가 일어나도 다른 종류의 알림은 정상 동작한다.
- 작업 서버: 메시지 큐에서 알림을 꺼내 제3자 서비스로 전달하는 역할을 담당하는 서버
- 제3자 서비스와 종류별 단말: 앞선 설계에 있는 것들

컴포넌트들이 어떻게 협력해서 알림을 전송하는지 살펴본다.

1. API를 호출하여 알림 서버로 알림을 보낸다.
2. 알림 서버는 사용자 정보, 단말 토큰, 알림 설정 같은 메타데이터를 캐시나 데이터베이스에서 가져온다.
3. 알림 서버는전송할 알림에 맞는 이벤트를 만들어서 해당 이벤트를 위한 큐에 넣는다. iOS 알림 이벤트는 iOS 알림 큐에 넣어야 한다.
4. 작업 서버는 메시지 큐에서 알림 이벤트를 꺼낸다.
5. 작업 서버는  알림을 제3자 서비스로 보낸다.
6. 제3자 서비스는 사용자 단말로 알림을 전송한다.

### 3단계 상세 설계

#### 안정성

분산 환경에서 알림 시스템을 설계할 때 안정성을 확보하기 위한 사항을 몇 가지를 반드시 고려한다.

##### 데이터 손실 방지

중요한 요구사항 중 하나로 어떤 상황에서도 알림이 소실되면 안 된다. 알림 지연이나 순서가 틀려도 되지만 사라지면 안된다는 뜻이다. 이 요구사항을 만족시키려면 알림 데이터를 데이터베이스에 보관하고 재시도 매커니즘을 구현해야 한다.
아래 그림과 같이 알림 로그 데이터베이스를 유지하는 것이 방법 중 하나이다.

<img width="298" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/1a4d7e79-45a1-4e42-96f9-e6fc1b1500d8">

##### 알림 중복 전송 방지

같은 알림이 여러 번 반복되는 걸 완전히 막을 수는 없다. 대부분 한번 전송되지만 분산 시스템의 특성상 같은 알림이 중복 전송되기도 한다. 그 빈도를 줄이려면 중복 탐지 매커니즘을 도입하고 오류를 처리해야 한다. 중복 방지 로직은 다음과 같이 생각해 볼 수 있다.

- 보내야 할 알림이 도착하면 이벤트 ID를 검사하여 이전에 보낸 이벤트인지 확인한다. 중복이면 버리고 아니면 발송한다.

#### 추가로 필요한 컴포넌트 및 고려사항

##### 알림 템플릿

알림 메시지 형식은 대부분 비슷하므로 템플릿을 써서 메시지를 매번 새로 만들 필요가 없게 해주면 좋다. 알림 템플릿은 인자나 스타일 추적 링크를 조정하기만 하면 사전에 지정한 형식에 맞춰 알람을 만들어내는 틀이다. 

아래는 간단한 예제이다.

```
BODY:
You dreamed of it. We dared it. [ITEM NAME] is back — only until [DATE].
CTA:
Order Now. Or, Save My [ITEM NAME]
```

템플릿을 사용하면 일관성을 유지할 수 있고, 오류 가능성과 동시에 알림 작성에 드는 시간을 줄일 수 있다.

##### 알림 설정

많은 앱에서는 알림 피로를 조절하기 위해 설정을 만들어 둔다. 이 정보는 알림 설정 테이블에 보관되며 다음과 같은 필드들이 필요하게 된다.

```
user_id bigInt
channel varchar # push notification, email or SMS
opt_in boolean # opt-in to receive notification
```

설정을 도입한 뒤에는 특정 알림을 보내기 전에 반드시 해당 사용자가 알림을 켜 두었는지 확인한다.

##### 전송률 제한

사용자에게 알림을 많이 보내지 않도록 하는 방법에는 알림의 빈도를 제한하는 것이다.
왜냐하면 너무 많이 보내면 사용자가 알림 기능을 꺼버릴 수 있기 때문이다.

##### 재시도 방법

제3자 서비스가 알림 전송에 실패하면, 해당 알림을 재시도 전용 큐에 넣는다.
같은 문제가 계속 발생하면 개발자에게 알림(alert)을 준다.

##### 푸시 알림과 보안

iOS, android 앱의 경우 알림 전송 API는 appKey와 AppSecret을 사용하여 보안을 유지한다.
따라서 인증된 혹은 승인된 클라이언트만 해당 API를 사용하여 알림을 보낼 수 있다.

##### 큐 모니터링

알림 시스템 모니터링에서 중요한 메트릭 중 하나는 큐에 쌓인 알림의 개수를 확인하는 것이다.
큐에 알림이 계속 쌓이는 갯수가 크다면 이벤트를 처리하고 있지 못하는 뜻이므로 작업 서버를 증설해야 한다는 신호이다.
아래 그래프가 그 사례이다.

<img width="265" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/8a4fa4b9-7ea2-4bc4-9bed-2e51c164b12e">

##### 이벤트 추적

데이터 분석 서비스는 보통 이벤트 추적 기능도 제공하므로 알림 시스템을 만들면 데이터 분석 서비스와도 통합해야만 한다.
아래 그림은 데이터 분석 서비스를 통해 추적하게 될 알림 시스템 이벤트 예시다.

<img width="429" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/1526cd86-d76a-42b5-8c1c-5b2d6aee76bf">

#### 수정된 설계안

여태까지의 설명을 모두 반영한 수정 설계안은 다음과 같다.

<img width="466" alt="image" src="https://github.com/jongfeel/BookReview/assets/17442457/a0e14901-f427-49be-9c21-ffba5b8c7b5f">

추가 컴포넌트 설명은 다음과 같다.

- 알림 서버에 인증(Authentication), 전송률 제한(Rate limit) 기능이 추가되었다.
- 전송 실패에 대응하기 위한 재시도 기능 추가. 전송에 실패한 알림은 큐에 넣고 지정된 횟수만큼 재시도한다.
- 전송 템플릿을 사용하여 알림 생성 과정을 단순화하고 내용의 일관성을 유지한다.
- 모니터링과 추적 시스템을 추가하여 시스템 상태를 확인하고 시스템을 개선하기 쉽도록 한다.

### 4단계 마무리

규모의 확장이 쉽고 푸시 알림, SMS, email 등 다양한 정보 전달 방식을 지원하는 알림 시스템을 알아 봤다.
컴포넌트 사이의 결합도를 낮추기 위해 메시지 큐를 적극적으로 사용하였다.
컴포넌트의 구현 방법과 최적화 기법에 대해서 아래 내용에 집중했다.

- 안정성(reliability): 메시지 전송 실패율을 낮추기 위해 안정적인 재시도 메커니즘을 도입
- 보안(security): 인증된 클라이언트만이 알림을 보낼 수 있도록 appKey, appSecret등의 매커니즘 이용
- 이벤트 추적 및 모니터링: 알림을 만들고 전송되기 까지의 과정을 추적하고 시스템 각 단계마다 이벤트를 추적하고 모니터링 할 수 있는 시스템 통합
- 사용자 설정: 사용자가 알림 수신 설정을 조절할 수 있도록 한다. 알림을 보내기 전에 확인하도록 설계를 변경
- 전송률 제한: 사용자에게 알림을 보내는 빈도를 제한한다.