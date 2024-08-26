# 쿼리 작성 및 최적화

## INSERT

### INSERT INGNORE

INSERT 문장의 IGNORE 옵션은 저장하는 레코드의 프라이머리 키나 유니크 인덱스 칼럼의 값이 이미 테이블에 존재하는 레코드와 중복되는 경우,
그리고 저장하는 레코드의 칼럼이 테이블의 칼럼과 호환되지 않는 경우 모두 무시하고 다음 레코드를 처리할 수 있게 해준다. 
  
주로 IGNORE 옵션은 여러 레코드를 하나의 INSERT 문장으로 처리하는 경우 유용하다.

```sql
INSERT IGNORE INTO salaries (emp_no, salary, from_date, to_date) VALUES
(10001, 60117, '1986-06-26', '1987-06-26'),
(10001, 62102, '1987-06-26', '1987-06-25'),
(10001, 66704, '1988-06-25', '1987-06-25'), 
(10001, 66596, '1989-06-25', '1987-06-25');
```
  
salaries 의 PK는 (emp_no, from_date)이므로 이미 중복되는 레코드가 있다면 에러가 발생한다. 하지만 MySQL 서버는
IGNORE 옵션이 있는 경우, 에러를 경고 수준의 메시지로 바꾸고 나머지 레코드의 INSERT를 계속 진행한다.

### INSERT...ON DUPLICATE KEY UPDATE

INSERT IGNORE 문장은 중복이나 에러 발생 건에 대해서는 모두 무시하겠지만 INSERT...ON DUPLICATE KEY UPDATE 문장은 PK나
유니크 인덱스의 중복이 발생하면 UPDATE 역할을 수행한다. 
  
비슷한 기능을 하는 REPLACE 문장은 내부적으로 DELETE와 INSERT 조합으로 작동하는데
InnoDB 에서는 DELETE와 INSERT 조합은 그다지 성능상 장점이 없기때문에 REPLACE보다는 INSERT...ON DUPLCATE KEY UPDATE 문장을 사용할 것을 권장한다.

```SQL
INSERT INTO daily_statistic (target_date, stat_name, stat_value)
VALUES (DATE(NOW()), 'VISIT', 1)
ON DUPLICATE KEY UPDATE stat_value=stat_value+1;
```

## 성능을 위한 테이블 구조

INSERT 문장의 성능은 쿼리 문장 자체보다는 테이블의 구조에 의해 많이 결정된다. 대부분 INSERT 문장은 단일 레코드를 저장하는 형태로 많이 사용되기 때문에
INSERT 문장 자체는 튜닝할 수 있는 부분이 별로 없는 편이다. 실제 쿼리 튜닝을 할 때도 소량의 레코드를 INSERT하는 문장 자체는 무시하는 경우가 많다.
  
### 대량 INSERT 성능

하나의 INSERT 문장으로 수백 건, 수천 건의 레코드를 INSERT한다면 `INSERT될 레코드를 PK값 기준으로 미리 정렬해서 INSERT 문장을 구성`하는 것이 성능에 도움이 될 수 있다.
  
INSERT하는 데이터가 PK값으로 정렬되어있지 않으면 LOAD DATA 문장이 `레코드를 INSERT할 때마다 InnoDB 스토리지 엔진은 PK를 검색해서
레코드가 저장될 위치`를 찾아야 한다. 그런데 정렬없이 랜덤하게 덤프된 데이터 파일의 경우 각 레코드의 PK가 너무 다른 값을 가지고 있다.
이로 인해 InnoDB 스토리지 엔진이 레코드를 저장할 때마다 PK의 B-Tree에서 이곳저곳 랜덤한 위치의 페이지를 메모리로 읽어와야 하기 때문에
처리가 더 느린 것이다. 물론 InnoDB 스토리지 엔진 버퍼 풀이 충분히 커서 인덱스가 모두 메모리에 적재돼 있다면 이보다는 차이가 조금 줄어들 수 있다.
  
반면 PK로 정렬된 데이터를 적재할 때는 다음에 INSERT할 레코드의 PK값이 이전에 INSERT된 값보다 항상 크기 때문에 메모리에는 PK의 마지막 페이지만 적재돼 있으면
새로운 페이지를 메모리로 가져오지 않아도 레코드를 저장할 위치를 찾을 수 있다.
  
세컨더리 인덱스 또한 SELECT 문장의 성능을 높이지만, 반대로 INSERT 성능은 떨어진다. 그래서 테이블에 세컨더리 인덱스가 많을 수록, 그리고 테이블이 클수록 INSERT 성능은 떨어진다.
테이블의 세컨더리 인덱스를 너무 남용하는 것은 성능상 좋지 않다. 물론 InnoDB 스토리지 엔진에서 세컨더리 인덱스의 변경은 일시적으로 체인지 버퍼에 버퍼링됐다가
백그라운드 스레드에 의해 일괄 처리될 수 있다. 하지만 너무 많은 세컨더리 인덱스는 백그라운드 작업의 부하를 유발하므로 전체적인 성능은 떨어진다.

### 프라이머리 키 선정

PK는 INSERT 성능을 결정하는 가장 중요한 부분이다. 테이블에 INSERT 되는 레코드가 PK 의 전체 범위에 대해 PK 순서와 무관하게 아주 랜덤하게 저장된다면
MySQL 서버는 레코드를 INSERT할 때마다 저장될 위치를 찾아야 한다. 즉 PK의 B-Tree 전체가 메모리에 적재돼 있어야 빠른 INSERT를 보장할 수 있는데 테이블의 크기가
크면 클수록 더 많은 메모리가 필요해진다. 일반적으로 메모리의 크기는 유한하며 디스크 크기보다 훨씬 용량이 작기 때문에 결국 어느 순간에는 저장될 위치를 찾기 위해서 디스크 읽기가 필요해진다.
  
InnoDB 스토리지 엔진을 사용하는 테이블의 PK는 클러스터링 키인데, 이는 세컨더리 인덱스를 이용하는 쿼리보다 PK를 이용하는 쿼리의 성능이 훨씬 빨라지는 효과를 낸다.
그래서 PK는 단순히 INSERT 성능만을 위해 설계해서는 안된다. PK 선정은 "INSERT 성능"과 "SELECT 성능"의 대립되는 두 가가지 요소 중에서 하나를 선택해야 함을 의미한다.
이 두가지 요소를 모두 만족하는 PK를 찾을 수 있다면 더할 나위 없이 좋겠지만, 그런 경우는 매우 드물다.
  
대부분 온라인 트랜잭션 처리를 위한 테이블들은 쓰기보다는 읽기 쿼리의 비율이 압도적으로 높다. 읽기는 거의 실행되지 INSERT가 매우 많이 실행되는 테이블이라면
PK 키를 단조 증가 또는 단조 감소하는 패턴의 값을 선택하는 것이 좋다. 주로 로그를 저장하는 테이블이 이런 류에 속한다.
  
하지만 상품이나 주문, 사용자 정보와 같이 중요 정보를 가진 테이블을들은 쓰기에 비해 읽기 비율이 압도적으로 높은 경우가 많다.
이런 류의 테이블에 대해서는 INSERT보다는 SELECT 쿼리를 빠르게 만드는 방향으로 PK를 선정해야 한다. 일반적으로 SELECT에 최적화된
PK는 단조 증가나 단조 감소 패턴과는 거리가 먼 경우가 많지만, 여전히 빈번하게 실행되는 SELECT 쿼리의 조건을 기준으로 PK를 선택하는 것이 좋다.
  
또한 SELECT는 많지 않고 INSERT가 많은 테이블에 대해서는 인덱스의 개수를 최소화하는 것이 좋다. 반면 INSERT는 많지 않고 SELECT가 많은 테이블에 대해서는
쿼리에 맞게 인덱스들을 추가해도 시스템 전반적으로 영향도가 크지 않다.

### Auto-Increment

InnoDB 스토리지 엔진을 사용하는 테이블은 자동으로 PK로 클러스터링된다. 즉 PK로 클러스터링되지 않게 InnoDB 테이블을 생성할 수 없다.
하지만 자동 증가 칼럼을 이용하면 클러스터링 되지 않는 테이블의 효과를 얻을 수 있다.
  
자동 증가 값을 PK로 해서 테이블을 생성하는 것은 MySQL에서 가장 빠른 INSERT를 보장하는 방법이다. 

많은 사용자가 매번 가장 최근에 저장된 Auto-Increment을 알기 위해 `SELECT MAX(member_id) FROM` 이런 쿼리를 만들지만
이렇게 MAX() 함수를 이용하는 방법은 상당히 잘못된 결과를 반환하기 때문에 다음과 같은 방법을 사용하는 것이 좋다.

```SQL
SELECT LAST_INSERT_ID();
```

MySQL에서는 현재 커넥션에서 가장 마지막에 증가된 Auto-Increment 값을 조회할 수 있게 LAST_INSERT_ID()라는 함수를 제공한다.
`SELECT MAX(member_id) FROM` 같은 쿼리르 상요하면 현재 커넥션 뿐만 아니라 다른 커넥션에서 증가된 Auto-Increment 값 까지 가져올 수 있으므로
주의해야 한다.

JDBC를 이용하는 경우 다음과 같이 별도의 SELECT 쿼리를 사용하지 않고도 자동 증가된 값을 가져올 수 있다.

```java
int affectedRowCount = stmt.executeUpdate("INSERT INTO ...", Statement.RETURN_GENERATED_KEYS);
ResultSet rs = stmt.getGeneratedKeys();
String autoInsertKey = (rs.next()) ? rs.getString(1) : null;
```

## Reference 

**위 글은 책 RealMySQL 8.0 2권을 구입하여 읽고 정리한 내용입니다.**
- [도서 홈페이지 https://wikibook.co.kr/realmysql802/](https://wikibook.co.kr/realmysql802/)



