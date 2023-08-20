[ToC]

## 2장 - 계략적인 규모 추정
### 2의 제곱수
- 그렇다. 2의 제곱수이다.
### 모든 프로그래머가 알아야 하는 응답지연 값
| 복붙했습니다...
- L1 cache reference 0.5 ns
- Branch mispredict 5 ns
- L2 cache reference 7 ns
- Mutex lock/unlock 100 ns
- Main memory reference 100 ns
- Compress 1K bytes with Zippy 10,000 ns
- Send 2K bytes over 1 Gbps network 20,000 ns
- Read 1 MB sequentially from memory 250,000 ns
- Round trip within same datacenter 500,000 ns
- Disk seek 10,000,000 ns
- Read 1 MB sequentially from network 10,000,000 ns
- Read 1 MB sequentially from disk 30,000,000 ns
- Send packet CA->Netherlands->CA 150,000,000 ns 

### 가용성에 관계된 수치들
- 가용성은 DevOps에서 많이 언급되는 용어
- 가용성을 올려야 할 것들
  - 알비언 서비스
  - GitLab
  - Elasticsearch
  - ...


## 함께 논의하고 싶은 주제
- 알비언 서비스의 가용성에 대하여 좀더 상세한 논의를 해야하지 않을까요?