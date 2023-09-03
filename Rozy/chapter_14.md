# chapter 14 유튜브 설계
 
## 시스템 설계 면접 포인트
- 주어진 시간 안에 적절한 기술을 골라 설계를 마치는 것
- 그 기술이 어떻게 동작하는지 상세히 설명하는 것은 중요하지 않음
- 무엇을 위해 어떤 저장소를 썻다 -> 이 정도만 설명해도 충분

## 비디오 업로드 절차

### 컴포넌트
- 메타데이터 데이터베이스: 비디오의 메타데이터 보관, 샤딩과 다중화 적용 가능
- 메타데이터 캐시: 성능을 높이기 위해 비디오 메타데이터와 사용자 객체 캐시
- 원본 저장소: 원본 비디오를 보관할 대형 이진 파일 저장소
- 트랜스코딩 서버: 비디오 트랜스코딩= 비디오 인코딩, 최적의 비디오 스트림을 제공하기 위해 비디오 포맷 변환 절차
- 트랜스코딩 비디오 저장소: 트랜스 코딩이 완료된 비디오 저장
- CDN: 비디오 캐시, 스트리밍은 CDN을 통해 이루어짐
- 트랜스코딩 완료 큐: 비디오 트랜스 코딩 완료 이벤트 보관
- 트랜스코딩 완료 핸들러: 트랜스코딩 완료 큐에서 이벤트 꺼내 메타데이터 캐시와 데이터베이스 갱신하는 작업 서버

### 메타데이터 갱신
- 원본 저장소에 파일이 업로드 되는 동안, 단말은 병렬적으로 비디오 메타데이터 갱신 요청을 API서버에 보냄
- API서버는 이 정보로 메타데이터 캐시와 데이터베이스를 업데이트

## 비디오 스트리밍 절차
- 스트리밍: 원격지의 비디오로부터 지속적으로 비디오 스트림을 전송 받아 영상을 재생하는 것
- 스트리밍 프로토콜: 비디오 스트리밍을 위해 데이터를 전송할때 사용
  - 프로토콜마다 지원하는 비디오 인코딩이 다르므로 잘 골라야 함
- 비디오는 CDN에서 바로 스트리밍됨-> 전송지연이 낮다

## 최적화
비디오 업로드와 스트리밍 최적화 방안

### 비디오 트랜스코딩
- 비디오가 다른 단말에서도 재생되도록 하기 위해 다른 단말과 호환되는 비트레이트와 포멧으로 저장되어야함
  - 비트레이트: 비디오를 구성하는 비트가 얼마나 빨리 처리되어야 하는지를 나타내는 단위
- 원본 비디오의 큰 저장공간 차지, 다른 단말과의 호환, 사용자의 네트워크 대역폭에 따른 화질 선택, 끊김없이 재생되도록 하기 위해 네트워크 상황에 따른 화질 변경 가능..
- 의 이유로 트랜스코딩은 중요하다

### 유향 비순환 그래프 모델 (DAG)
- : 작업을 단계별로 배열할 수 있도록 하여 해당 작업들이 순차적으로 or 병렬적으로 실행될 수 있도록 하는것
- 검사, 비디오 인코딩, 섬네일, 워터마크 등 작업을 나누어 실행될 수 있도록 한다

### 비디오 트랜스코딩 아키텍처

### 속도 최적화
- 비디오 병렬 업로드
  - 하나의 비디오를 작은 GOP들로 분할하여 업로드
  - 일부가 실패하도 빠르게 업로드 재개 가능
- 업로드 센터를 사용자 근거리에 지정
  - 업로드 센터를 여러곳에 둔다
  - CDN을 업로드 센터로 이용
- 모든 절차를 병렬화
  - 느슨하게 결홉된 시스템을 만들어 병렬성을 높이기
  - 메시지 큐를 도입하여 앞의 작업이 끝나기를 기다릴 필요없이 큐에 보관된 이벤트 각각을 병렬적으로 처리 가능

### 안정성 최적화
- 미리 사인된 업로드 URL
  - 허가받은 사용자만이 비디오를 업로드할 수 있도록 미리 사인된 업로드 URL을 사용
  - 클라이언트가 HTTP서버에 post요청하여 미리 사인된 URL을 받는다
- 비디오 보호
  - 저작권을 보호하기 위해 디지털 저작권 권리, AES암호화, 워터마크 사용

### 비용 최적화
- 인기 비디오는 CDN을 통해 재생하되 다른 비디오는 비디오 서버를 통해 재생
- 시청 패턴 분석 중요


# 논의주제
최적화 방법 중 병렬화가 포인트인 것 같습니다. 경험해보셨던 병렬화된 절차가 뭐가 있으셨나요? 다양한 예시를 들어보고 싶습니다.
