# 8. 인덱스

- 인덱스는 쿼리 성능에 아주 중요한 키워드다.
- 각 인덱스의 특성과 차이는 중요하다.

## 8.1 디스크 읽기 방식

- 디스크는 컴퓨터에서 가장 느린 부분이다.
- 디스크 I/O를 줄이는게 성능 튜닝의 관건이다.

### 8.1.1 HDD vs SSD

- HDD : 데이터 저장용 플래터 (원판)
- SSD: 플래시 메모리 → 원판을 회전시킬 필요가 없어 HDD보다 훨신 빠르다.
- 순차 I/O 에서는 SSD가 HDD보다 조금 빠르거나 비슷하다.
- 랜덤 I/O 에서는 SSD가 훨신 빠른 성능을 보여준다.
    - DBMS에서 랜덤 I/O 작업의 비중이 크므로  SSD가 좋은 선택이 된다.

### 8.12 랜덤 I/O와 순차 I/O

- 3개의 페이지를 디스크에 기록할때
    - 순차 I/O는 1번의 시스템 콜을
    - 랜덤 I/O는 3번의 시스템 콜을 날린다.
- 원판이 있는 HDD에서는 거의 3배 느리다고 볼 수 있다.
- SSD에서는  순차 I/O와 랜덤 I/O의 차이가 없을것으로 예측되지만 실제로는 그렇지 않다.
- 쿼리를 튜닝해서 랜덤  I/O를 순차 I/O로 바꿔 실행할 방법은 그다지 많지 않다.
    - 쿼리를 튜닝하는 것은 랜덤 I/O 자체를 줄여주는 것이 목적이다.
        
        → 꼭 필요한 데이터만 읽도록 개선하는것
        
- 인덱스 레인지 스캔은 데이터를 읽기 위해 주로 랜덤 I/O를 사용하고 풀 테이블 스캔은 순차 I/O를 사용한다.
    - 그래서 큰 테이블의 레코드 대부분을 읽는 작업에서는 인덱스를 사용하지 않고 풀테이블 스캔을 사용하도록 유도할 때도 있다. (순차 I/O)가 훨신 빨리 많은 레코드를 읽어올 수 있기 때문에)

## 8.2 인덱스란?

- 책의 찾아보기(색인) 처럼 원하는 결과를 빠르게 가져오기 위해 칼럼의 값고 해당 레코드가 저장된 주소를 키와 값의 쌍으로 만들어야한다.
- 책의 색인처럼 미리 정렬되어 있다.
- 정렬되어 있어 빠르게 탐색할 수 있지만 삽입, 삭제, 수정 문장의 속도가 느리다.
    - 결론적으로 인덱스는 데이터의 저장 (INSERT, UPDATE, DELETE)의 성능을 희생하고 데이터의 읽기 속도를 높이는 기능이다.
    - 인덱스를 하나 더 추가할지 말지 고민 → 저장속도를 어디까지 희생할 수 있는지, 읽기 속도를 얼마나 더 빠르게 만들어야 하는지에 대한 고민

### 데이터를 관리하는 방식에 의한 분류

- 프라이머리 키
    - 레코드를 대표하는 칼럼의 값으로 만들어진 인덱스를 의미한다.
    - 식별자라고 부른다.
    - NULL을 허용하지 않고 중복을 허용하지 않는다.
- 세컨더리 인덱스
    - 프라이머리 키를 제외한 모든 인덱스다.

### 저장 방식별로 구분

- B-Tree 알고리즘
    - 가반 일반적으로 사용되는 인덱스
    - 칼럼의 값을 변형하지 않고 원래의 값을 이용해 인덱싱 한다.
- ### Hash 인덱스
    - 해시값을 계산해서 인덱싱한다.
    - 매우 빠른 검색을 지원한다.
    - 값을 변형해서 인덱싱하므로 값의 일부만 검색하거나 범위를 검색할 때는 해시인덱스를 사용할 수 없다.
    - 주로 메모리 기반의 데이터베이스에서 많이 사용한다.

### 데이터의 중복여부로 분류

- 유니크 인덱스
    - 동등조건 (Equal, =) 으로 검색한다는 것은 항상 1건의 레코드만 찾으면 더 찾지 않아도 된다.
- 유니크하지 않은 인덱스

## B(Balanced)-Tree 인덱스

- 가장 범용적인 목적으로 사용되는 인덱스 알고리즘이다.
- 주로 B+Tree, B*Tree로 변형되어 사용된다. (뒤의 그림과 설명을 보니 MySQL인덱스는 B+Tree로 구현된것 같음)

### 구조 및 특성

- 루트 노드 : 최상위 노드
- 리프 노드 : 최하위 노드
- 브랜치 노드 : 루트 노드도 아니고 리프 노드도 아닌 중간 노드

- 프라이머리 키
    - 대부분의 DBMS에서인덱스의 키값은 모두 정렬돼 있지만, 데이터 파일의 레코드는 정렬돼 있지 않고 임의의 순서로 저장돼 있다.
    - InnoDB 테이블에서 레코드는 클러스터되어 디스크에 저장되므로 기본적으로 프라이머리 키 순서로 정렬되어 저장된다.
    - MyISAM의 “레코드 주소”는 MyISAM 테이블에서 INSERT된 순번이거나 데이터 파일 내의 위치다. (생성 옵션에 따라 달라짐)
    - InnoDB는 프라이머리 키가 ROWID의 역할을 한다.
    
- 세컨더리 인덱스
    - MyISAM의 세컨더리 인덱스는 물리적 주소(레코드 주소)를 갖는다.
    - InnoDB의 세컨더리 인덱스는 물리적 주소가 아닌 논리적 주소(프라이머리 키)를 가진다.
        
        → 프라이머리키를 주소처럼 사용하기 때문에
        
    - 따라서 세컨더리 인덱스를 타고 가서 프라이머리 키를 얻은 후 프라이머리 인덱스를 타고가 실제 레코드를 읽는다.

### 8.3.2 인덱스 키 추가 및 삭제

- 인덱스 키 추가
    - B-Tree에 키를 저장할때 적절한 위치를 검색후 리프노드에 저장된다.
    - 리프노드가 꽉차서 더이상 저장 불가능하면 리프노드가 분리되고 그 상위 브랜치 노드에 추가해 작업한다.
    - 이런 이유로 B-Tree는 상대적으로 키 추가 하는 비용이 많이 든다.
        - 대략적으로 레코드를 추가하는 작업의 비용이 1이라하면 인덱스에 키를 추가하는 작업 비용을 1.5라고 본다.
    - 이 시간의 대부분은 디스크로부터 인덱스 페이지를 읽고 쓰기를 하는 시간이다.
    - InnoDB 스토리지 엔진은 인덱스 키 추가 작업을 지연시켜 나중에 처리할 수 있다.
        - 하지만 프라이머리 키나 유니크 인덱스의 경우 중복 체크가 필요해 즉시 B-Tree에 추가하거나 삭제한다. (체인지 버퍼)

- 인덱스 키 삭제
    - 키값이 삭제되는 경우 리프 노드를 찾아 그냥 삭제 마크만 하면 작업이 완료된다. (이 공간은 나중에 재활용할 수 있다.)
    - 이 작업또한 디스크 쓰기가 필요하다.
    - 5.5버전 이상에서는 삭제작업도 버퍼링되어 지연처리될 수 있다.

- 인덱스 키 변경
    - 삭제 후 삽입된다.
    - 이 작업도  체인지 버퍼를 통해 지연 처리 될 수 있다.

- 인덱스 키 검색
    - 인덱스의 존재 이유다. (삽입, 삭제, 검색을 느리게 하는 대신 검색을 빠르게)
    - 루트 노드부터 시작해 브랜치 노드를 거쳐 최종 리프노드까지 이동하면서 비교작업을 수행한다. (트리 탐색)
    - 인덱스 트리탐색은 SELECT 뿐 아니라 UPDATE나 DELETE를 처리하기 위해서도 해야한다.
    - B-Tree 인덱스를 이용한 검색은 100%일치 또는 값의 앞부분만 일치하는 경우, 부등호 비교조건에서 활용할 수 있다.
    - 뒷부분만 일치하는 경우나 인덱스의 키 값에 변형이 가해진 후(함수 계산 후) 비교되는 경우 B-Tree의 빠른 검색 기능을 사용할 수 없다.

### 8.3.3 B-Tree 인덱스 사용에 영향을 미치는 요소

- 칼럼의 크기, 레코드의 건수, 유니크한 인덱스 키 값의 개수 등이 있다.
- 인덱스 키 값의 크기
    - 인덱스 페이지(또는 블록) : InnoDB가 디스크에 데이터를 저장하는 기본단위
        - 디스크의 모든 읽기 및 쓰기 작업의 최소 작업 단위가 된다.
        - InnoDB의 버퍼 풀에서 데이터를 버퍼링하는 기본 단위다.
        - 루트, 브랜치, 리프 노드를 구분한 기준이다.
    - InnoDB 스토리지 엔진의 페이지 크기와 키 값의 크기에 따라 결정된다.
        - 페이지 크기 (innodb_page_size : 4KB~64KB 가능) : 기본 16KB
        - 인덱스 키의 크기 : 16B 로 가정
        - 자식 노드의 주소 크기 (페이지 별로 다름 6B ~ 12B) : 12B로 가정
        - 하나의  페이지에 저장할 수 있는 키의 갯수 : (16 * 1000) / (16 + 12) ⇒ 약 585개
        - 쿼리가 레코드 500개를 읽어야 한다면 인덱스 페이지 한번으로 해결 가능하다.
    - 인덱스 키값의 크기가 커지면 디스크로부터 읽어야 하는 횟수가 늘어나고, 그만큼 느려진다.
    - 인덱스 키값의 크기가 커지면 메모리에 캐시해둘 수 있는 레코드 수가 줄어들어 메모리의 효율이 떨어진다.
- B-Tree 깊이
    - depth가 3인 B-Tree의 경우
        - 키 값의 크기가 16바이트인 경우 : 최대 2억개의 키 값을 담을 수 있다.
        - 키 값의 크기가 32바이트인 경우 : 최대 5천만개의 키 값을 담을 수 있다.
    - B-Tree의 깊이는 MySQL에서 값을 검색할 때 몇번이나 랜덤하게 디스크를 읽어야 하는지에 직결된다.
    - 인덱스 키 값의 크기를 작게 만들면 깊이가 줄어든다.
- 선택도(기수성)
    - 모든 인덱스 키 값 가운데 유니크한 값의 수를 의미한다.
    - 인덱스 키 값 가운데 중복된 값이 많아지면 기수성은 낮아지고 선택도도 낮아진다.
    - 인덱스는 선택도가 높을수록 검색대상이 줄어들어 빠르게 처리된다.
        - (항상 검색에만 인덱스가 사용되는게 아니고 정렬이나 그루핑 같은 작업을 위해 인덱스를 만드는것이 나은 경우도 많다.)
    - 예시 : tb_test 테이블의 전체 레코드 개수는 1만 건이다.
    - MySQL에서는 인덱스의 통계 정보(유니크한 값의 개수)가 관리되기 때문에 city 칼럼의 기수성은 작업 범위에 아무런 영향을 미치지 못한다.(??)
        
        `SELECT * FROM tb_test WHERE country=’KOREA’ AND city=’SEOUL’;`
        
        - A : country 칼럼의 유니크한 값의 개수가 10개다.
            - 평균적으로 A 케이스의 경우에는 1,000건 (1만 / 10) 개가 조회된다.
                
                → 1건의 레코드를 위해 쓸모없는 999건의 레코드를 더 읽은것 → 비효율적
                
        - B : country 칼럼의 유니크한 값의 개수가 1000개다.
            - 평균적으로 B 케이스의 경우에는 10건 (1만 / 1,000) 개가 조회된다.
                
                → 1건의 레코드를 위해 쓸모없는 9건의 레코드를 더 읽은것 → 효율적
                
- 읽어야 하는 레코드의 건수
    - 인덱스를 거치지 않고 바로 테이블의 레코드를 읽는 예상 비용 : 1이라 가정하면
    - 인덱스를 통해 테이블의 레코드를 읽는 예상 비용 : 4~5
    - 옵티마이저가 인덱스를 통해 읽어야할 레코드의 건수를 전체 테이블 레코드의 20~25%를 넘게 예상하면 테이블을 모두 직접 읽어서 필요한 레코드만 가려내는 방식으로 처리하는 것이 효율적이다.
    

### 8.3.4 B-Tree 인덱스를 통한 데이터 읽기

**인덱스 레인지 스캔**

- 가장 대표적인 접근 방식, 가장 빠르다.
- 예시 `SELECT * FROM employees WHERE first_name BETWEEN ‘Ebbe’ AND ‘Gad’;`
    - 인덱스 레인지 스캔은 검색하야 할 인덱스의 범위가 결정됐을 때 사용하는 방식이다.
    - 일단 시작해야 할 위치를 찾으면(인덱스 탐색 : index seek) 그때부터 리프노드의 레코드만 순서대로 읽으면 된다. (차례로 쭉) (인덱스 스캔 : index scan)
    - 리프노드의 끝까지 읽으면 리프 노드 간의 링크를 이용해 다음 리프 노드를 찾아서 다시 스캔한다.
    - 스캔을 멈춰야할 위치까지 다다르면 지금까지 읽은 레코드를 사용자에게 반환하고 쿼리를 끝낸다.
    - 어떠한 방향(정순, 역순)으로 읽든 인덱스를 구성하는 칼럼의 정순 또는 역순으로 정렬된 상태로 레코드를 가져온다. (레코드 인덱스 키가 이미 정렬되어 있어서)
    - 인덱스의 리프 노드에서 검색조건에 일치하는 건들은 데이터 파일에서 레코드를 읽어오는 과정이 필요하다.
    - 이때 랜덤  I/O가 발생한다. (인덱스를 통해 데이터 레코드를 읽는 작업은 비용이 많이단다.)
    - 쿼리가 필요로 하는 데이터에 따라 index scan에서 읽은 레코드 주소를 이용해 저장된 페이지를 가져와 최종 레코드를 읽는 과정이 필요 없다.
        - Handler_read_key : index seek 이 실행된 횟수
        - Handler_read_next, Handler_read_prev : index scan에서 읽은 레코드 수
        - Handler_read_first, Handler_read_last : MIN(), MAX() 와 같이 제일 큰, 작은 값만 읽는 경우 증가하는 상태값 → 인덱스만 index scan 후 레코드는 하나만 읽으면 된다.

**인덱스 풀 스캔**

- 인덱스의 처음부터 끝까지 모두 읽는다.
- 대표적으로 인덱스 (A, B, C) 칼럼의 순서로 만들어져 있는데 쿼리의 조건절이 B나 C로 검색을 하는 경우다.
- 일반적으로 인덱스의 크기는 테이블의 크기보다 작다. 직접 테이블을 처음부터 끝까지 읽는 것보다는 인덱스만 읽는것이 효율적이다.
- 쿼리가 인덱스에 명시된 칼럼만으로 조건을 처리할 수 있는 경우 주로 이 방식이 사용된다.
- 인덱스 레인지 스캔보다 빠르지는 않지만 테이블 풀 스캔보다는 효율적이다.

**루스(Loose) 인덱스 스캔**

- 오라클의 “인덱스 스킵 스캔”과 작동방식이 비슷하다.
- 루스 인덱스 스캔과 반대되는 방식으로 앞의 두 방법을 타이트 인덱스 스캔이라 부른다.
- 레인지 스캔과 비슷하게 작동하지만 중간에 필요하지 않은 인덱스 키 값은 무시하고 다음으로 넘어간다.
- 일반적으로 `GROUP BY`, `MAX()`, `MIN()` 함수에 대해 최적화를 하는 경우에 사용된다.

**인덱스 스킵 스캔**

- 데이터베이스 서버에서 인덱스의 핵심은 값이 정렬돼 있다는 것이다. → 인덱스를 구성하는 칼럼의 순서가 매우 중요하다.
- 예시 : 인덱스 = (gender, birth_date)
    1. `SELECT * FROM employees WHERE gender='M' AND birth_date >= ‘1965-02-01’`
        - 쿼리를 효율적으로 사용할 수 있다.
    2. `SELECT * FROM employees WHERE birth_date ≥ ‘1965-02-01’`
        - gender 칼럼에 대한 비교 조건이 없는 첫 번째 쿼리는 인덱스를 사용 할 수 없었다.
        - 이런 이유로 인덱스 스킵 스캔 최적화 기능이 도입
            - 8.0부터 옵티마이져가 gender 칼럼을 건너 뛰어서 birth_date 칼럼만으로도 인덱스 검색이 가능하게 해줌
        - 만약 인덱스 스킵 스캔 옵션을 제거한다면
            - 인덱스 풀 스캔을 사용해야함 → 비효율적 →  풀 테이블 스캔을 사용
        - 인덱스 스킵 스캔을 사용한다면
            - `SELECT gender, birth_date FROM employees WHERE birth_date ≥ ‘1965-02-01’`
            - gender 칼럼에서 유니크한 값을 모두 조회해서 주어진 쿼리에 gender 칼럼의 조건을 추가해서 쿼리를 다시 실행하는 형태로 처리한다.
                - `SELECT gender, birth_date FROM employees WHERE gender='M' AND birth_date >= ‘1965-02-01’`, `SELECT gender, birth_date FROM employees WHERE gender='F' AND birth_date >= ‘1965-02-01’` 이렇게 두개의 쿼리를 실행하는것과 비슷한 형태의 최적화를 한다.
        - 하지만 8.0에 새로 도입된 기능이라 단점이 있다.
            - WHERE 조건절에 조건이 없는 인덱스의 선행 칼럼의 유니크한 값의 개수가 적어야함
                - 레인지 스캔 시작지점을 검색하는 작업이 많이 필요해져 쿼리의 성능이 매우 떨어진다.
            - 쿼리가 인덱스에 존재하는 칼럼만으로 처리 가능해야함
                - gender, birth_date 만 조회하는게 아니라 모든 칼럼을 조회한다면 풀 테이블 스캔을 실행 하는게 낫다.

### 8.3.5 다중 칼럼(Multi-column) 인덱스

- 서비스용 데이터베이스에서는 2개 이상의 칼럼을 포함하는 인덱스가 더 많이 사용된다.
- Concatenated Index 라고도 한다.
- 다중 칼럼 인덱스에서는 인덱스 내에서 각 칼럼의 위치가 상당히 중요하다.

### 8.3.6 B-Tree 인덱스의 정렬 및 스캔 방향

- 인덱스를 읽는 방향에 따라 정렬이 달라진다.
- 어느 방향에서 읽을지는 옵티마이져가 실시간으로 만들어 낸다.

**인덱스의 정렬**

8.0버전 부터는 각 컬럼의 정렬을 오름차순 또는 내림차순으로 설정할 수 있다.

- 인덱스 스캔 방향
    - 정순으로 읽으면 → 오름차순 정렬 효과
    - 역순으로 읽으면 → 내림차순 정렬 효과
    - MIN(), MAX()도 읽는 방향을 조정해서 하나만 읽어올 수 있다.
- 내림차수 인덱스
    - 오름차순 인덱스 : 작은 값의 인덱스 키가 B-Tree의 왼쪽으로 정렬된 인덱스
    - 내림차순 인덱스 : 큰 값의 인덱스 키가 B-Tree의 왼쪽으로 정렬된 인덱스
    - 인덱스 정순 스캔 : 인덱스 키의 크고 작음에 관계없이 인덱스 리프 노드의 왼쪽 페이지부터 오른쪽으로 스캔
    - 인덱스 역순 스캔 : 인덱스 키의 크고 작음에 관계없이 인덱스 리프 노드의 오른쪽 페이지부터 왼쪽으로 스캔
    
    ```sql
    SELECT * FROM t1 ORDER BY tid ASC LIMIT 12619775,1; # 4.15 sec 
    SELECT * FROM t1 ORDER BY tid DESC LIMIT 12619775,1; # 5.35 sec
    ```
    
    - 위 쿼리들은 LIMIT 때문에 풀 테이블 스캔을 한다.
    - 결과는 tid 값이 가장 큰 레코드 1건을 가져온다.
    - 정순으로 스캔하는 1번 쿼리가 더 빠르다. 이유는
        - 페이지 잠금이 인덱스 정순 스캔에 적합한 구조다.
        - 페이지 내에서 인덱스 레코드가 단방향으로만 연결된 구조
    - 따라서 자주 역순으로 읽어야 할때는 인덱스를 내림차순으로 설정하는것이 고려되야한다.
    - ORDER BY … DESC 쿼리가 소량의 레코드에 드물게 실행되는 경우라면  어떤 정렬이든 크게 상관없다.

### B-Tree 인덱스의 가용성과 효율성

- 쿼리의 WHERE, GROUP BY, ORDER BY 절이 어떤 경우에 인덱스를 사용할 수 있고 어떤 방식으로 사용하는지 알아야한다.

**비교조건의 종류와 효율성**

- `SELECT * FROM dept_emp WHERE dept_no=’d002’ AND emp_no emp_no >= 10114;`
- 케이스 A : INDEX(dept_no, emp_no)
    - dept_no=’d002’ AND emp_no≥’10144’인 레코드를 찾고 dept_no가 ‘d002’가 아닐때까지 쭉 읽으면 된다. → 상당히 효율적
- 케이스 B : INDEX(emp_no, dept_n,)
    - emp_no≥’10144’ AND dept_no=’d002’ 인 레코드를 찾고 그 모든 레코드에 대해 dept_no가 ‘d002’ 인지 비교하는 과정을 거쳐야 한다. → 비효율적
- dept_no는 비교 작업의 범위를 좁히는데 아무런 도움을 주지 못한다. → ‘작업 범위 결정 조건’
    
    → 작업 범위 결정 조건이 많으면 쿼리의 성능이 높아진다.
    
- emp_no는 비교 작업의 범위를 좁히는 데 도움을 준다. → ‘필터링 조건’, ‘체크 조건’
    
    → 체크 조건이 많다고 해서 쿼리의 처리 성능을 높이지 못한다. 오히려 느리게 만든다.
    

**인덱스의 가용성**

- 케이스 A : INDEX(first_name)
    - 이 경우 `WHERE first_name LIKE ‘%mer’` 로는 검색할 수 없다.
- 케이스 B : INDEX(dept_no, emp_no)
    - emp_no 값으로만 검색하면 인덱스를 효율적으로 사용할 수 없다.

**가용성과 효율성 판단**

- B-Tree 인덱스를 사용할 수 없는 경우
    - NOT-EQUAL로 비교된 경우
    - LIKE “%승환 ” 같이 뒷부분이 일치하는 문자열 패턴으로 비교된 경우
    - 스토어드 함수나 드란 연산자로 인덱스 칼럼이 변형된 후 비교된 경우
    - NOT-DETERMINISTIC 속성의 스토어드 함수가 비교조건에 사용된 경우 (?)
    - 데이터 타입이 서로 다른 비교
    - 문자열 데이터 타입의 콜레이션이 다른경우
    
    `INDEX ix_test (column_1, column_2, column_3, .. column_n)`
    
    - 작업 범위 결정 조건으로 인덱스를 사용하지 못하는 경우
        - column_1 칼럼에 대한 조건이 없는 경우
        - column_1 칼럼의 비교 조건이 위의 인덱스 사용 불가 조건 중
- 작업 범위 결정 조건으로 인덱스를 사용하는 경우
    - column_1 ~ column_i 까지 동등 비교 형태로 비교
    - column_i 칼럼에 대해 다음 연산자중 하나로 비교
        - 동등 비교
        - 크다 작다 형태
        - LIKE로 좌측 일치 패턴

## 8.8 클러스터링 인덱스

- 클러스터링 : 여러개를 하나로 묶는다.
- 테이블의 레코드를 비슷한 것들끼리 묶어서 저장하는 형태로 구현된다.
    - 주로 비슷한 값들을 동시에 조회하는 경우가 많기 때문에
- InnoDB에서만 지원한다.

### 8.8.1 클러스터링 인덱스

- 프라이머리 키에 대해서만 적용되는 내용이다.
    - 프라이머리 키 값이 비슷한 레코드끼리 묶어서 저장하는것이다.
- 프라이머리 키값에 의해 레코드의 저장 위치가 결정된다는 것이다.
    - 프라이머리 키가 변하면 그 레코드의 물리적인 저장 위치가 바뀌어야 한다.
- 일반 B-Tree와 구조가 비슷하지만 세컨더리 인덱스를 위한 B-Tree의 리프 노드와 달리 인덱스의 리프 노드에는 레코드이 모든 칼럼이 같이 저장돼 있다.
- 프라이머리키가 없는 InnoDB 테이블은 InnoDB가 다음 우선순위대로 프라이머리 키를 대체할 칼럼을 선택한다.
    1. 프라이머리 키가 있으면 기본적으로 프라이머리 키를 클러스터링 키로 선택
    2. NOT NULL 옵션의 유니크 인덱스 중에서 첫 번째 인덱스를 클러스터링 키로 선택
    3. 자동으로 유니크한 값을 가지도록 증가되는 칼럼을 내부적으로 추가한 후, 클러스터링 키로 선택
        - 자동으로 추가된 이 일련번호 칼럼은 사용자에게 노출되지 않으며 쿼리 문장에 명시적으로 사용될 수 없다. → 꼭 프라이머리 키를 명시적으로 생성하자.

### 8.8.2 세컨더리 인덱스에 미치는 영향

- MyISAM, MEMORY 테이블 : 레코드를 INSERT후 절대 위치를 이동하지 않음
    - 실제 레코드가 저장된 위치가 ROWID 역할을 함
    - 세컨더리 인덱스가 ROWID를 이용해 실제 데이터 레코드를 찾아올 수 있음
    - 레코드의 주소를 확인한 후, 레코드의 주소를 이용해 최종 레코드를 가져옴
- InnoDB : 레코드가 위치를 이동할 수 있음
    - 클러스터링 키 값이 변경될때마다 데이터 레코드의 주소가 변경 → 그때마다 해당 테이블의 모든 인덱스에 저장된 주솟값을 변경해야함.
    - 이런 이유로 세컨더리 인덱스는 레코드가 저장된 주소가 아니라 프라이머리 키값을 저장하도록 구현됨
    - 세컨더리 인덱스를 검색해 레코드의 프라이머리 키 값을 확인한 후, 프라이머리 키 인덱스를 검색해서 최종 레코드를 가져옴
    

### 8.8.3 클러스터링 인덱스의 장점과 단점

**장점**

- 프라이머리 키로 검색할 때 처리 성능이 매우 빠름 (특히 범위 검색)
- 테이블의 모든 세컨더리 인덱스가 프라이머리 키를 가지고 있어 인덱스만으로 처리될 수 있는 경우가 많음

**단점**

- 테이블의 모든 세컨더리 인덱스가 클러스터링 키를 갖기 때문에 클러스터링 키 값의 크기가 클 경우 전체적으로 인덱스의 크기가 커짐
- 세컨더리 인덱스를 통해 검색할때 프라이머리 키로 다시 한번 검색해야 하므로 처리 성능이 느림
- INSERT 할때 프라이머리 키에 의해 레코드의 저장 위치가 결정되기 때문에 처리 성능이 느림
- 프라이머리 키를 변경할 때 레코드를 DELETE 하고 INSERT 하는 작업이 필요하기 때문에 처리 성능이 느림

### 8.8.4 클러스터링 테이블 사용 시 주의사항

- 클러스터링 인덱스 키의 크기
    - 모든 세컨더리 인덱스가 프라이머리 키값을 포함하기 때문에 프라이머리 키의 크기가 커지면 세컨더리 인덱스 크기는 급격히 증가한다.
- 프라이머리 키는 AUTO-INCREMENT 보다는 업무적인 칼럼으로 생성 (가능한 경우)
    - 프라이머리 키 = 저장되는 위치 이다.
    - 대부분 검색에서 상당히 빈번하게 사용되므로 해당 레코드를 대표할 수 있다면 그 칼럼을 프라이머리 키로 지정하자.
- 프라이머리 키는 반드시 명시하자.
- AUTO-INCREMENT 칼럼을 인조 식별자로 사용할경우
    - 여러개의 컬럼이 복합으로 프라이머리 키가 만들어지는 경우
        - 프라이머리 키의 크기가 길어질 때가 가끔 있다.
            - 프라이머리 키의 크기가 길어도 세컨더리 인덱스가 필요하지 않으면 그대로 프라이머리 키를 사용하는 것이 좋다.
            - 세컨더리 인덱스도 필요하고 프라이머리 키의 크기도 길다면 AUTO_INCREMENT 칼럼으 추가하고 이를 프라이머리 키로 설정한다.
            - 로그 테이블과 같이 조회보다 INSERT 위주의 테이블은 인조식별자가 성능향상에 유리하다.
            

## 8.9 유니크 인덱스

- 유니크 인덱스는 인덱스보다 제역 조건에 가깝다.
    - 인덱스에 같은 값이 2개 이상 저장될 수 없다.
    - NULL은 특정값이 아니라 2개 이상 저장될 수 있다.

### 유니크 인덱스와 일반 세컨더리 인덱스의 비교

- 구조상으로는 아무 차이점이 없다.

**인덱스 읽기**

- 유니크 인덱스는 실행계획이 달라 빠르게 동작하는것이지 인덱스의 특성 때문에 빠른것이 아니다.

**인덱스 쓰기**

- 유니크 인덱스의 키 값을 쓸 때는 중복된 값이 있는지 없는지 확인하는 과정이 한 단계 더 필요하다. → 더 느림
- 중복된 값을 체크할때는 읽기 잠금을 사용하고, 쓰기를 할때는 쓰기 잠금을 사용하는데 이과정에서 데드락이 아주 빈번히 발생한다.
- 중복 체크를 해야하기 때문에 체인지 버퍼에 버퍼링 하지 못한다.

### 유니크 인덱스 사용 시 주의사항

- 꼭 필요한 경우라면 생성하는 건 당연
- 하지만 성능이 더 좋아질 것으로 생각하고 불필요하게 만드는것 좋지 않다.

## 8.10 외래 키

- InnoDB 스토리지 엔진에서만 생성할 수 있다.
- 외래키 제약이 설정되면 자동으로 연관되는 테이블의 칼럼에 인덱스까지 생성된다.
- 외래키가 제거되지 않은 상태에서는 자동으로 생성된 인덱스를 삭제할 수 없다.

- 외래키 관리에 필요한 두가지 특징
    - 테이블의 변경(쓰기 잠금)이 발생하는 경우에만 잠금 경합(잠금 대기)이 발생한다.
        - 부모 테이블이 변경 작업중에 자식 테이블이 외래키 칼럼(부모의 키)을 변경하는 경우 부모 테이블이 작업을 마칠때 까지 대기한다.
        - 자식 테이블에서 외래키 칼럼을 변경중에 부모 테이블에서 해당 칼럼을 수정하려 하면 자식 테이블을 기다린다.
    - 외래키와 연관되지 않은 칼럼의 변경(쓰기 잠금)은 최대한 잠금 경합(잠금 대기)을 발생시키지 않는다.