# 인덱스

## 디스크 읽기 방식

디스크 같은 기계식 장치의 성능은 CPU나 메모리에 비해 상당히 느리게 발전해왔다. 데이터베이스 성능 튜닝은 어떻게 디스크 I/O를 줄이는지
관건일 때가 상당히 많다.
  
컴퓨터에서 CPU나 메모리같은 주요 장치는 대부분 전자식 장치지만 HDD는 기계식 장치다. 그래서 데이터베이스 서버에서는 항상
디스크 장치가 병목이 된다. 
  
SSD는 기존 HDD에서 데이터 저장용 플래터를 제거하고, 그 대신 플래시 메모리를 장착하고 있어 원판을 회전시킬 필요가 없으므로 아주 빨리
데이터를 읽고 쓸 수 있다. 
  
디스크의 헤더를 움직이지 않고 한 번에 많은 데이터를 읽는 `순차 I/O`에서 SSD가 HDD보다 조금 빠르거나 거의 비슷한 성능을 보이기도 한다.
하지만 SSD의 가장 큰 장점은 기존 HDD보다 `랜덤 I/O`가 훨씬 빠르다는 것이다.
  
데이터베이스 서버에서 `순차 I/O` 작업은 그다지 비중이 크지 않다. `랜덤 I/O`를 통해 작은 데이터를 읽고 쓰는 작업이 대부분이므로
SSD의 장점은 DBMS용 스토리지에 최적이라고 볼 수 있다.

### 랜덤 I/O

`랜덤 I/O`라는 표현은 HDD의 플래터를 돌려서 읽어야 할 데이터가 저장된 위치로 디스크 헤더를 이동시키고 데이터를 읽는 것을 말한다.
그런데 사실 `순차 I/O`도 이 작업 과정은 같다.

### 그럼 어떤 차이?

`순차 I/O`는 3개의 페이지를 디스크에 기록하기 위해 1번의 시스템 콜을 요청한다. 그러나 `랜덤 I/O`는 3개의 페이지를 디스크에 기록하기 위해
3번의 시스템 콜을 요청하는 것에 차이가 있다.

보통 디스크에 데이터를 읽고 쓰는데 걸리는 시간은 **디스크 헤더를 움직여서 읽고 쓸 위치로 옮기는 단계에서 결정**된다.
  
즉 디스크의 성능은 디스크 헤더의 위치 이동 없이 얼마나 많은 데이터를 한 번에 기록하느냐에 의해 결정된다고 볼 수 있다. 그래서 
여러번 쓰기 또는 읽기를 요청하는 `랜덤 I/O` 작업이 부하가 훨씬 더 크다.

결국 이 때문에 MySQL 서버에서는 `랜덤 I/O`를 줄이기 위해 그룹 커밋이나 바이너리 로그, InnoDB 로그 버퍼등의 기능을 내장하고
`순차 I/O` 비중을 늘려 성능을 향상시키는 것이다.

다만 쿼리를 튜닝해서 `순차 I/O` 비중을 높일 방법은 그다지 많지 않고, 쿼리를 처리할 때 
꼭 필요한 데이터만 읽도록 해서 `랜덤 I/O` 자체를 줄이는 것이 튜닝의 주 목적이다.  

## Reference 

**위 글은 책 RealMySQL 8.0을 구입하여 읽고 정리한 내용입니다.**
- [도서 홈페이지 https://wikibook.co.kr/realmysql801/](https://wikibook.co.kr/realmysql801/)