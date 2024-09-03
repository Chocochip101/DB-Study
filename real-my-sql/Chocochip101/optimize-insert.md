일반적으로 온라인 트랜잭션 서비스에서 INSERT 문장은 대부분 1건 또는 소량의 레코드를 INSERT하는 형태이므로 그다지 성능에 대해서 고려할 부분이 많지 않다. 오히려 많은 INSERT 문장이 동시에 실행되는 경우 INSERT 문장 자체보다는 테이블의 구조가 성능에 더 큰 영향을 미친다. 하지만 많은 경우 INSERT의 성능과 SELECT의 성능을 동시에 빠르게 만들 수 있는 테이블 구조는 없다. 그래서 INSERT와 SELECT 성능을 어느 정도 타협하면서 테이블 구조를 설계해야 한다. 여기서는 INSERT 문장의 주의사항 몇 가지와 테이블의 용도에 따라 테이블 구조를 선택하는 방법을 간단히 살펴보겠다. 

# 고급 옵션 
SELECT 문장만큼 다양하진 않지만 INSERT 문장에도 사용할 수 있는 유용한 기능이 있다. 여기서는 대표적으로 INSERT IGNORE 옵션과 INSERT ON DUPLICATE KEY UPDATE 옵션을 살펴보겠다. **두 옵션 모두 유니크 인덱스나 프라이머리 키에 대해 중복 레코드를 어떻게 처리할지를 결정**한다. 그리고 INSERT INOGRE 옵션은 추가로 INSERT 문장의 에러 핸들링에 대한 기능도 포함하고 있다. 
## INSERT IGNORE 
INSERT 문장의 IGNORE 옵션은 저장하는 레코드의 프라이머리 키나 유니크 인덱스 칼럼의 값이 이미 테이블에 존재하는 레코드와 중복되는 경우, 그리고 저장하는 레코드의 칼럼이 테이블의 칼럼과 호환되지 않는 경우 **모두 무시하고 다음 레코드를 처리할 수 있게 해준다**. 주로 IGNORE 옵션은 다음과 같이 여러 레코드를 하나의 INSERT 문장으로 처리하는 경우 유용하다.

테이블의 프라이머리 키는 중복되는 레코드가 있다면 해당 레코드는 에러가 발생한다. 하지만 MySQL 서버는 IGNORE 옵션이 있는 경우, 에러를 경고 수준의 메시지로 바꾸고 나머지 레코드의 INSERT를 계속 진행한다. INSERT하고자 하는 데이터가 정교하지 않아도 되는 경우 INSERT를 실행하기 전에 레코드 건건이 중복 체크를 실행하지 않고 INSERT IGNORE 명령으로 처리하는 방식으로 자주 사용된다. INSERT하는 테이블이 프라이머리 키와 유니크 인덱스를 동시에 가지고 있는 경 우 INSERT IGNORE는 두 인덱스 중 하나라도 중복이 발생하는 레코드에 대해서는 INSERT를 무시(IGNORE) 한다. INSERT IGNORE 옵션은 단순히 유니크 인덱스의 중복뿐만 아니라 데이터 타입이 일치하지 않아서 INSERT 를 할 수 없는 경우에도, 다음 예제와 같이 칼럼의 기본 값으로 INSERT를 하도록 만들기도 한다. 이 경우에는 NOT NULL 칼럼에 NULL이 입력되어 NULL 대신 숫자 칼럼의 기본 값인 0을 INSERT를 한 것이다.

프로그램 코드에서 중복을 무시하기 위 해 INSERT IGNORE 옵션을 사용한다면 데이터 중복 이외의 에러가 발생할 여지가 없는지 면밀히 확인한 후 적용하는 것이 좋다. **제대로 검증되지 않은 INSERT IGNORE 문장은 의도하지 않은 에러까지 모두 무시 해버릴 수도 있다.** 

## INSERT ... ON DUPLICATE KEY UPDATE 
INSERT IGNORE 문장은 중복이나 에러 발생 건에 대해서는 모두 무시하겠지만 **INSERT ON DUPLICATE KEY UPDATE 문장은 프라이머리 키나 유니크 인덱스의 중복이 발생하면 UPDATE 문장의 역할을 수행하게 해준다.** MySQL 서버의 REPLACE 문장도 INSERT ON DUPLICATE KEY UPDATE 문장과 비슷한 역할을 하지만 내부적으로 REPLACE 문장은 DELETE와 INSERT의 조합으로 작동한다. 하지만 INSERT ... ON DUPLICATE KEY UPDATE 문장은 **중복된 레코드가 있다면 기존 레코드를 삭제하지 않고 기존 레코드의 칼럼을 UPDATE하는 방식으로 작동**한다. INSERT ON DUPLICATE KEY UPDATE는 다음 예제와 같이 일별로 집계되는 값을 관리할 때 편리하게 사 용할 수 있다.

위 예제의 daily_statistic 테이블의 프라이머리 키는 (target_date, stat_name) 조합으로 생성돼 있기 때문에 일별로 stat_name은 하나씩만 존재할 수 있다. 그래서 특정 날짜의 stat_name이 최초로 저장되는 경우에는 INSERT 문장만 실행되고, ON DUPLICATE KEY UPDATE 절 이하의 내용은 무시된다. 이미 레코드가 존재한다면 INSERT 대신 ON DUPLICATE KEY UPDATE 절 이하의 내용이 실행된다. 예제 쿼리에서는 이미 해 당 날짜의 집계 레코드가 존재한다면 기존 레코드의 stat_value 칼럼의 값에 1을 더한 값을 UPDATE한다. 다음 예제는 배치 형태로 GROUP BY된 결과를 daily_statistic 테이블에 한 번에 INSERT하는 예제다. 위의 예제에서 stat_value 칼럼에는 GROUP BY 결과 건수를 저장하고 있다. 그런데 이 쿼리에서 이미 해 당 일자와 stat_name이 동일한 레코드가 있었다면 ON DUPLICATE KEY UPDATE 절이 실행될 것이다. 그런데 ON DUPLICATE KEY UPDATE 절에서는 GROUP BY 결과인 "COUNT(*) "를 참조할 수 없다. 그래서 예제 쿼리에서 는 "Invalid use of group function" 에러가 발생한 것이다. 이러한 경우 VALUES() 함수를 사용하면 된 다. 우선 VALUES 함수로 사용한 쿼리 예제를 먼저 살펴보자.

VALUES() 함수는 칼럼명을 인자로 사용하는데, 위의 예제와 같이 VALUES(stat_value) 라고 사용하면 MySQL 서버는 인자로 주어진 stat_ *value 칼럼에 INSERT하려고 했던 값을 반환한다. 그래서 INSERT . SELECT GROUP BY 문장에서 실제 저장하려고 했던 값이 무엇인지 몰라도, VALUES(stat_value) 함수를 사용하면 stat_value 칼럼에 INSERT하고자 했던 값을 다시 가져올 수 있다. 앞의 예제 쿼리에서는 stat* value 칼럼에 INSERT하려고 했던 값과 기존 레코드의 stat_ value 칼럼이 가지고 있던 값의 합을 stat_ value 칼럼에 업데이트한다. MySQL 8.0.20 버전부터는 VALUES() 함수를 사용하면 다음과 같은 경고 메시지가 출력된다. Warning (Code 1287): 'VALUES function' is deprecated and will be removed in a future release. Please use an alias (INSERT INTO · VALUES (…) AS alias) and replace VALUES(col) in the DUPLICATE KEY UPDATE clause with alias.col instead MySQL 8.0.20 이후 버전에서는 VALUES() 함수가 지원되지 않을 예정이므로(Deprecated) 다음과 같 은 문법으로 대체해서 사용할 것을 권장한다. INSERT SELECT : 형태의 문법이 아닌 경우에는 다음 예제와 같이 INSERT되는 레코드에 대해 별칭 을 부여해서 참조하는 문법을 사용하면 VALUES() 함수의 사용을 피할 수 있다.

# LOAD DATA 명령 주의 사항 
일반적으로 RDBMS에서 데이터를 빠르게 적재할 수 있는 방법으로 LOAD DATA 명령이 자주 소개된다. MySQL 서버의 LOAD DATA 명령도 내부적으로 MySQL 엔진과 스토리지 엔진의 호출 횟수를 최소화하고 **스토리지 엔진이 직접 데이터를 적재하기 때문에** 일반적인 INSERT 명령과 비교했을 때 매우 빠르다고 할 수 있다. 하지만 MySQL 서버의 LOAD DATA 명령은 다음과 같은 단점이 있다. 

- 단일 스레드로 실행 
- 단일 트랜잭션으로 실행 

LOAD DATA 명령으로 적재하는 데이터가 아주 많지 않다면 앞의 2가지 사항은 그다지 큰 문제가 되지 않는다. 하지만 데이터가 매우 커서 실행 시간이 아주 길어진다면 다른 온라인 트랜잭션 쿼리들의 성능이 영향을 받을 수 있다. 우선 LOAD DATA 명령은 단일 스레드로 실행되기 때문에 적재해야 할 데이터 파일이 매우 크다면 시간이 매우 길어질 수 있다. 테이블에 여러 인덱스가 있다면 LOAD DATA 문장이 레코드를 INSERT하고 인덱스에도 키 값을 INSERT해야 한다. 그런데 테이블에 레코드가 INSERT되면 될수록 테이블과 인덱스의 크기도 커지게 된다. 하지만 LOAD DATA 문장은 단일 스레드로 실행되기 때문에 시간이 지나면 지날수록 INSERT 속도는 현저히 떨어진다. 또한 LOAD DATA 문장은 하나의 트랜잭션으로 처리되기 때문에 LOAD DATA 문장이 시작한 시점부터 언두 로그(Undo Log)가 삭제되지 못하고 유지돼야 한다. 이는 언두 로그를 디스크로 기록해야 하는 부하를 만들기도 하지만, 언두 로그가 많이 쌓이면 레코드를 읽는 쿼리들이 필요한 레코드를 찾는 데 더 많은 오버혜드를 만들어 내기도 한다.

가능하다면 LOAD DATA 문장으로 적재할 데이터 파일을 하나보다는 여러 개의 파일로 준비해서 LOAD DATA 문장을 동시에 여러 트랜잭션으로 나뉘어 실행되게 하는 것이 좋다. 테이블 간 데이터 복사 작업 이라면 LOAD DATA 문장보다는 INSERT SELECT 문장으로 WHERE 조건 절에서 데이터를 부분적으로 잘라서 효율적으로 INSERT할 수 있게 해주는 것이 좋다. 실제 테이블 간의 복사라면 LOAD DATA 문장으로 데이터를 잘라서 실행하기는 번거롭지만 INSERT SELECT 문장으로는 프라이머리 키 값을 기준 으로 데이터를 잘라서 여러 개의 스레드로 실행하기가 훨씬 용이하다. 
# 성능을 위한 테이블 구조 
INSERT 문장의 성능은 쿼리 문장 자체보다는 **테이블의 구조에 의해 많이 결정**된다. 대부분 INSERT 문장은 단일 레코드를 저장하는 형태로 많이 사용되기 때문에 INSERT 문장 자체는 튜닝할 수 있는 부분이 별로 없는 편이다. 실제 쿼리 튜닝을 할 때도 소량의 레코드를 INSERT하는 문장 자체는 무시하는 경우가 많다. 

## 대량 INSERT 성능 
하나의 INSERT 문장으로 수백 건, 수천 건의 레코드를 INSERT한다면 INSERT될 레코드들을 프라이머리 키 값 기준으로 **미리 정렬해서 INSERT 문장을 구성하는 것이 성능에 도움이 될 수 있다**.

LOAD DATA 문장이 레코드를 INSERT할 때마다 InnoDB 스토리지 엔진은 프라이머리 키를 검 색해서 레코드가 저장될 위치를 찾아야 한다. 그런데 **정렬없이 랜덤하게 덤프된 데이터 파일의 경우에 는 각 레코드의 프라이머리 키가 너무 다른 값을 가지고 있**다. 이로 인해 InnoDB 스토리지 엔진이 레코드를 저장할 때마다 프라이머리 키의 B-Tree에서 이곳저곳 랜덤한 위치의 페이지를 메모리로 읽어와야 하기 때문에 처리가 더 느린 것이다. 물론 InnoDB 스토리지 엔진의 버퍼 풀이 충분히 커서 테이블의 인덱스가 모두 메모리에 적재돼 있다면 이보다는 차이가 조금 줄어들 수도 있다. 반면 프라이머리 키로 정렬된 데이터 파일을 적재할 때는 다음에 INSERT할 레코드의 프라이머리 키 값이 직전에 INSERT된 값보다 항상 크기 때문에 메모리에는 프라이머리 키의 마지막 페이지만 적재돼 있으면 새로운 페이지를 메모리로 가져오지 않아도 레코드를 저장할 위치를 찾을 수 있다. 

물론 INSERT 성능은 프라이머리 키 정렬 여부로 많이 결정되지만 프라이머리 키가 전부는 아니다. 테이블의 세컨더리 인덱스는 SELECT 문장의 성능을 높이지만, 반대로 INSERT 성능은 떨어진다. 그래서 테이블에 세컨더리 인덱스가 많을수록, 그리고 테이블이 클수록 INSERT 성능은 떨어진다. **세컨더리 인덱스도 정렬된 순서대로 INSERT될 수 있다면 더 빠른 성능을 얻을 수 있다**. 하지만 하나의 테이블에서 모든 세컨더리 인덱스가 저장되는 순서대로 정렬되게 보장하기는 어렵다. **이러한 이유로 테이블의 세컨더리 인덱스를 너무 남용하는 것은 성능상 좋지 않다.** 물론 InnoDB 스토리지 엔진에서 세컨더리 인덱스의 변경은 일시적으로 체인지 버퍼에 버퍼링됐다가 백그라운드 스레드에 의해 일괄 처리될 수 있다. 하지만 너무 많은 세컨더리 인덱스는 백그라운드 작업의 부하를 유발하므로 전체적인 성능은 떨어진다.

## 프라이머리 키 선정 
이미 살펴본 바와 같이 테이블의 프라이머리 키는 INSERT 성능을 결정하는 가장 중요한 부분이다. 테이블에 INSERT되는 레코드가 프라이머리 키의 전체 범위에 대해 프라이머리 키 순서와 무관하게 아주 랜덤하게 저장된다면 MySQL 서버는 레코드를 INSERT할 때마다 저장될 위치를 찾아야 한다. 즉 **프라이머리 키의 B-Tree 전체가 메모리에 적재돼 있어야 빠른 INSERT를 보장할 수 있는데, 테이블의 크기가 크면 클수록 더 많은 메모리가 필요해진다.** 일반적으로는 메모리의 크기는 유한하며 디스크 크기보다 용량이 작기 때문에 결국 어느 순간에는 저장될 위치를 찾기 위해서 디스크 읽기가 필요해진다. InnoDB 스토리지 엔진을 사용하는 테이블의 프라이머리 키는 클러스터링 키인데, 이는 세컨더리 인덱스를 이용하는 쿼리보다 프라이머리 키를 이용하는 퀴리의 성능이 훨씬 빨라지는 효과를 낸다. 그래서 프라이머리 키는 단순히 INSERT 성능만을 위해 설계해서는 안 된다. **프라이머리 키의 선정은 "INSERT 성능"과 "SELECT 성능"의 대립되는 두 가지 요소 중에서 하나를 선택해야 함을 의미**한다. 이 두 가지 요소를 모두 만족하는 프라이머리 키를 찾을 수 있다면 더할 나위 없이 좋다. 하지만 그런 경우는 매우 드물다. 대부분 온라인 트랜잭션 처리를 위한 테이블들은 쓰기(INSERT, UPDATE, DELETE)보다는 읽기(SELECT) 쿼 리의 비율이 압도적으로 높다. **SELECT는 거의 실행되지 않고 INSERT가 매우 많이 실행되는 테이블이라면 테이블의 프라이머리 키를 단조 증가 또는 단조 감소하는 패턴의 값을 선택하는 것이 좋다**. 주로 로그를 저장하는 테이블이 이런 류에 속한다. 하지만 상품이나 주문, 사용자 정보와 같이 중요 정보를 가 진 테이블들은 쓰기에 비해 읽기 비율이 압도적으로 높은 경우가 많다. 이러한 류의 테이블에 대해서는 INSERT보다는 SELECT 쿼리를 빠르게 만드는 방향으로 프라이머리 키를 선정해야 한다. 일반적으로 SELECT에 최적화된 프라이머리 키는 단조 증가나 단조 감소 패턴과는 거리가 먼 경우가 많지만, 여전히 빈번하게 실행되는 SELECT 쿼리의 조건을 기준으로 프라이머리 키를 선택하는 것이 좋다. 또한 SELECT는 많지 않고 INSERT가 많은 테이블에 대해서는 인덱스의 개수를 최소화하는 것이 좋다. 반 면 INSERT는 많지 않고 SELECT가 많은 테이블에 대해서는 쿼리에 맞게 필요한 인덱스들을 추가해도 시 스템 전반적으로 영향도가 크지 않다. 물론 SELECT가 많은 테이블에 대해서는 자연적으로 세컨더리 인덱스가 많아진다. 

응용 프로그램을 개발하고 유지 보수를 하면서 기능을 계속 추가하다 보면 수많은 테이블을 생성하게 된다. 이 많은 테이블에 대해서 모든 성능 요소를 고려해서 설계한다는 것은 매우 어려운 작업이 될 수 있다. 다행히도 응용 프 로그램에서 사용하는 모든 테이블에 대해서 이런 고민을 할 필요는 없다. 예를 들어, 만들어야 할 테이블이 100개라면 그중에서 대량의 레코드를 가질 것으로 예상되는 테이블의 수는 10%도 안 되는 경우가 많다. **테이블을 설계할 때 1~2 백만 건 또는 그 이하의 레코드를 가지는 테이블에 대해 너무 많은 시간을 소모하지 않아도 된다**. 테이블이 이미 작기 때문에 아무리 튜닝을 잘해도 얻을 수 있는 성능 효과가 크지 않기 때문이다. 물론 레코드 건수가 많지 않더라도 쿼리 가 매우 빈번하게 실행된다면 튜닝에 신중을 기해야 한다. 소량의 테이불에 투자할 시간을 대량의 테이블에 투자해서 대용량의 테이블에 대한 쿼리들이 최적으로 작동할 수 있게 하자. 

## Auto-Increment 칼럼 
SELECT보다는 INSERT에 최적화된 테이블을 생성하기 위해서는 다음 두 가지 요소를 갖춰 테이블을 준비하면 된다. 

- 단조 증가 또는 단조 감소되는 값으로 프라이머리 키 선정 
- 세컨더리 인덱스 최소화

InnoDB 스토리지 엔진을 사용하는 테이블은 자동으로 프라이머리 키로 클러스터링된다. 즉, 프 라이머리 키로 클러스터링되지 않게 InnoDB 테이블을 생성할 수 없다. 하지만 자동 증가(Auto Increment) 칼럼을 이용하면 클러스터링되지 않는 테이블의 효과를 얻을 수 있다. 
```sql
CREATE TABLE access_10g( id BIGINT NOT NULL AUTO_INCREMENT, ip_address INT UNSIGNED, uri VARCHAR(200), ... visited_at DATETIME, PRIMARY KEY(id) ) 
```
위와 같이 자동 증가 값을 프라이머리 키로 해서 테이블을 생성하는 것은 MySQL 서버에서 가장 빠른 INSERT를 보장하는 방법이다. 물론 세컨더리 인덱스가 하나도 없는 테이블이라면 금상첨화일 것이다. 

MySQL 서버에서는 자동 증가 값의 채번을 위해서는 잠금이 필요한데, 이를 AUTO-INC 잠금이라고 한다. 그리고 이 잠금을 사용하는 방식을 변경할 수 있게 innodb_autoinc_lock_mode라는 시스템 변수를 제공한다. 이 설정은 InnoDB 이외의 스토리지 엔진을 사용하는 테이블에는 영향을 미치지 않는다.

- innodb_autoinc_lock_mode = 0: 항상 AUTO-INC 잠금을 걸고 한 번에 1씩만 증가된 값을 가져온다. 이는 MySQL 5.1 버전의 자동 증가 값 채번 방식인데, 단순히 이전 버전과의 호환성 및 성능 비교 테스트 용도로만 사용 하기 위해서 남겨 둔 것이다. 서비스용 MySQL 서버에서는 이 방식을 사용할 필요가 없다. 
- innodb_autoinc_lock_mode = 1 (Consecutive mode): 단순히 레코드 한 건씩 INSERT하는 쿼리에서는 AUTO INC 잠금을 사용하지 않고 뮤텍스를 이용해 더 가볍고 빠르게 처리한다. 하지만 하나의 INSERT 문장으로 여러 레코 드를 INSERT하거나 LOAD DATA 명령으로 INSERT하는 쿼리에서는 AUTO-INC 잠금을 걸고 필요한 만큼의 자동 중 가 값을 한꺼번에 가져와서 사용한다. INSERT 순서대로 채번된 자동 증가 값은 일관되고, 자동 증가 값은 연속된 번 호를 갖게 된다(그래서 이 모드의 이름이 "Consecutive mode' "인 것이다). 
- innodb_autoinc_lock_mode = 2 (Interleaved mode): LOAD DATA나 벌크 INSERT를 포함한 INSERT 계열의 문 장을 실행할 때 더 이상 AUTO-INC 잠금을 사용하지 않는다. 이때 자동 증가 값을 적당히 미리 할당받아서 처리할 수 있으므로 가장 빠른 방식이다. 이 모드에서 채번된 번호는 단조 증가하는 유니크한 번호까지만 보장하며, INSERT 순서와 채번된 번호의 연속성은 보장하지 않는다. 하나의 INSERT 문장으로 저장된 레코드라고 하더라도 중간에 번호가 띄엄띄엄 발급될 수도 있다. 그래서 쿼리 기반의 복제 (SBR, Statement Based Replication)를 사용하는 MySQL에서는 소스 서버와 레플리카 서버의 자동 증가 값이 동 기화되지 못할 수도 있으므로 주의해야 한다. 

MySQL 5.7 버전까지는 innodb_autoinc_lock_mode 시스템 변수의 기본값은 1이었지만 MySQL 8.0 버전부터는 기본값이 2로 변경됐다. 이는 MySQL 8.0 버전부터 복제의 바이너리 로그 포맷 기본 값이 STATEMENT에서 ROW로 변경됐기 때문이다. MySQL 서버의 버전과 관계없이, 복제를 STATEMENT 바이너리 로그 포맷으로 사용 중이라면 innodb_autoinc_lock_mode 또한 1로 설정해야 한 다는 사실을 잊지 말자. 자동 증가 값이 반드시 연속이어야 한다면 innodb_autoinc_lock_mode를 2보다는 1로 설정하는 것이 좋다. 하지만 innodb_autoinc_lock_mode를 0이나 1로 설정하더라도 시간이 조금씩 지나면서 연속된 값에 빈 공간이 생길 가능성이 높다. 그러므로 자동 증가 값이 반드시 연속이어야 한다는 요건에 너무 집착 하지는 말자. 

> AUTO_INCREMENT는 지금까지 살펴본 일련번호를 칼럼에 저장하는 기능뿐만 아니라 가장 최근에 저장된 AUTO_ INCREMENT 값을 안전하게 조회할 수 있는 기능도 있다. 하지만 많은 사용자가 매번 "SELECT MAX(member_id) FROM "과 같은 쿼리를 실행해 최댓값을 조회한다. 이렇게 MAX() 함수를 이용하는 방법은 상당히 잘못된 결과를 반환할 수도 있으므로 다음에서 살펴볼 방법을 사용하는 것이 좋다. MySQL에서는 현재 커넥션에서 가장 마지막에 증가된 AUTO_INCREMENT 값을 조회할 수 있게 LAST_INSERT_ID()라 는 함수를 제공한다. 위의 예제는 AUTO_INCREMENT 값을 증가시키는 INSERT 쿼리 문장을 실행하고, 그 이후에 LAST_ INSERT_ID()라는 함수를 실행하면 가장 마지막에 사용된 AUTO_INCREMENT 값을 반환하는 것을 보여준다. "SELECT MAX(...) . - 같은 쿼리를 사용하면 현재 커넥션뿐만 아니라 다른 커넥션에서 증가된 AUTO_INCREMENT 값까 지 가져올 수 있으므로 사용하지 않는 것이 좋다. 하지만 LAST_INSERT_ID() 함수는 다른 커넥션에서 더 큰 AUTO_ INCREMENT 값을 INSERT했다고 하더라도 현재 커넥션에서 가장 마지막으로 INSERT된 AUTO_INCREMENT 값만 반환하 기 때문에 안전하다. 그리고 JDBC를 이용하는 경우에는 다음과 같이 별도의 SELECT 쿼리를 사용하지 않고도 자동 증가된 값을 가져올 수 있다.