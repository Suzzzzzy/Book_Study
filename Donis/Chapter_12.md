# 채팅 시스템 설계
- 요구사항 : DAU  5천만명, 채팅방 최대 입장 가능 인원 100명, 1:1 채팅, 그룹 채팅, 사용자 접속상태, 텍스트 메시지 전송(100,000자 길이 제한), 메시지 이력 영구 저장, 다양한 단말기 지원. 동시 접속 지원


- 개략적 설계안 : 풀링, 롱 풀링, 웹소켓
  - 개략적 설계안 : 무상태 서비스, 상태 유지 서비스, Tirth Party Push, 규모 확장성, 저장소


- 상세 설계
  - 서비스 탐색, 메시지 흐름, 접속상태 표시


## 논의 주제 : 서비스 탐색(Service Discovery)에 대해서 어떻게 이해하고 계신가요?
