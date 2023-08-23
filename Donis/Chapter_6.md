# 키-값 저장소 설계
- CAP 정리
  - 일관성(Consistency), 가용성(Availability), 파티션 감내(partition tolerance) 3가지 요구사항 모두를 만족은 불가함.
    - CP : 가용성을 희생
    - AP : 데이터 일관성을 희생
    - CA : 존재하지 않는 시스템
  - 시스템 컴포넌트
    - 데이터 파티션 : 안정 해시를 사용한 서버들의 부하 분산
    - 데이터 다중화(replication) : 데이터 센터 다중화
    - 데이터 일관성(consistency)
      - 정족수 합의 프로토콜(Quorum Consensus)
      - 일관성 모델(Consistency model) : 강한 일관성(xtrong Consistency), 약한 일관선(weak consistency), 최종 일관성(eventual consistency)
    - 일관성 불일치 해소(inconsistency resolution) : 데이터 버저닝(versioning), 벡터 시계(vector clock)
    - 장애 처리 : 머클 트리(Merkle tree) = 해시 트리(Hash tree) 10억(1B) 개의 키를 백만(1M)개의 버킷으로 관리하는 것. 하나의 버킷은 1,000개의 키를 관리.
    - 시스템 아키텍처 다이어그램
    - 쓰기 경로(write path) : SSTable(Sorted-String Table)
    - 읽기 경로(read path) : 블룸 필터(Bloom Filter)
  ## 논의 주제 : 벡터 시계가 잘 이해가 안됩니다...
