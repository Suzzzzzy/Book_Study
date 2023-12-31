[ToC]

## 8장 - URL 단축기 설계
### 1단계 - 문제 이해 및 설계 범위 확정
기본적 기능
1. URL 단축: 주어진 긴 URL을 훨씬 짧게 줄임
2. URL 리디렉션(redirection): 축약된 URL로 HTTP 요청이 오면 원래 URL로 안내
3. 높은 가용성과 규모 확장성

### 2단계 - 개력적 설계안 제시 및 동의 구하기
#### API 엔드포인트
두개의 엔드포인트롤 필요로 함
1. URL 단축용 엔드포인트
  - 인자: {longUrl: longURLstring}
  - 반환: 단축 URL
2. URL 리디렉션용 엔드포인트
  - 반환: HTTP 리디렉션 목적지가 될 원래 URL

#### URL 리디렉션

#### URL 단축

### 3단계 상세 설계
#### 데이터모델
- 모든 것을 해시 테이블에 두면 메모리는 유한해서 비쌈
- <단축 URL, 원래 URL> 순서쌍을 둔 관계형 데이터베이스 저장하는게 더 좋음

#### 해시 함수
- 해식 값 길이: 0-9, a-z, A-Z -> 62개
- 해시 함수 구현에 쓰이는 두 기술
  - 해시 충돌 해소
    - 문자열 7자로 줄이는 해시 함수
      - CRS32
      - MD5
      - SHA-1
  - base-62 변환
    - 62진법으로 사용
  - 두개 비교하자면...
    - 135쪽 참고

#### URL 단축시 상세 설계
1. 입력
2. DB 확인
3. DB에 있으면 URL 반환
4. 없으면 ID 생성
5. 생성된 ID를 단축 URL로 변환
6. ID, 단축 URL을 DB에 저장


#### URL 리디렉션 상세 설계
로드밸런서의 동작 흐름
1. 단축 URL 클릭
2. 웹 서버에 전달
3. 캐시에 있으면 URL을 바로 클라이언트에 전달
4. 없으면 데이터베이스에서 조회
5. 캐시에 넣은후 클라이언트에 전달

### 4단계 마무리
추카포카

## 함께 논의하고 싶은 주제
- tiny URL을 만드는 이유가 무엇인가요?? 이 장에서 그 설명이 빠진것 같습니다. 단지 시스템 설계의 복잡성만 늘어난다는 기분이 듭니다. API 요청이 저렇게 길어진다면 POST로 주로 처리했던것 같아서 더 와닿지가 않네요. tiny url이라는 것도 처음 들어봤고요. 실재로 써보신적이 있을까요?

