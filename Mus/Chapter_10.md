[ToC]

## 10장 - 알림 시스템 설계
### 1단계 - 문제 이해 및 설계 범위 확정

### 2단계 - 개략적 설계안 제시 및 동의 구하기
- 알림 유형별 지원 방안
- 연락처 정보 수집 절차
- 알림 전송 및 수신 절차

#### 알림 유형별 지우너 방안
- IOS
  - 알림 제공자
    - 단말 토큰: 알림 고유 식별자
    - 페이로드: 알림 내용을 담은 JSO 딕셔너리
  - [APNS](./Glossary.md#10장)
  - iOS 단말: 사용자 스맡폰
- 안드로이드 푸시 알림
   - FCM(Firebase Cloud Messaging)
- SMS 메시지
  - 제3자 서비스 많이 씀
- 이메일
  - 상용 이메일 서비스 많이 씀
    - 메일침프 써봤어요 되게 좋아요

#### 연락처 정보 수집 절차
DB에 잘 저장

#### 알림 전송 및 수신 절차
- 개략적인 설계안(초안)
  - 여러개의 서비스
  - 알림 시스템
    - 알림 전송을 위한 API 제공
    - 제 3자 서비스에 전달할 알림 페이로드를 만들어야 함
  - 제 3자 서비스
    - 확장성이 중요
    - 제한사항 확인
  - iOS, 안드로이드 SMS, 이메일 단말
  - 문제점
    - SPOF(Single-Point-Of-Failure): 서버가 장애나면 서비스 장애로 이어짐
    - 규모 확장성
- 개략적 설계안(개선된 버전)
  - 데이터베이스와 캐시 알림 시스템의 주 서버에서 분리(!!)
  - 수평적 규모 확장
  - 시스템 컴포넌트 사이의 강한 결합을 끊음

### 3단계 - 상세 설계
- 안정성
- 추가로 필요한 컴포넌트 및 고려사항
  - 템플릿
  - 설정
  - 전송률 제한
  - 재시도 메커니즘
  - 보안
- 개선될 설계안

#### 안정성
- 데이터 손실 방지
- 알림 중복 전송 방지
  - 중복 전송을 100프로 방지하는 것은 불가???

#### 추가로 필요한 컴포넌트 및 고려사항
- 알림 템플릿
- 알림 설정
- 전송률 제한
- 재시도 방법
- 푸시 알림과 보안
- 이벤트 추적
  - 알림 확인률
  - 클릭율
  - 실제 앱 사용..

#### 수정된 설계안

### 4단계 마무리
- 안정성
- 보안
- 이벤트 추적 및 모니터링
- 사용자 설정
- 전송률 제한


## 함께 논의하고 싶은 주제
- 중복 전송을 100% 방지하는게 어렵다는데, 중복으로 받은 경험이 있는지 혹은 중복으로 보내는 시스템이 만든 경험이 있는지 궁금합니다.
