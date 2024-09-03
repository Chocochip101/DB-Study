# 쿼리 작성 및 최적화

## SELECT

### 서브쿼리

쿼리를 작성할 때 서브쿼리를 사용하면 단위 처리별로 쿼리를 독립적으로 작성할 수 있다.
조인처럼 여러 테이블을 섞어 두는 형태가 아니어서 쿼리의 가독성도 높아지며, 복잡한 쿼리도 손쉽게 작성할 수 있다.
8.0 버전부터는 서브쿼리 처리가 많이 개선됐다.
  
서브 쿼리는 여러 위치에서 사용될 수 있다. 대표적으로 SELECT 절과 FROM 절, WHERE 절이다.
사용되는 위치에 따라 쿼리의 성능 영향도와 MySQL 서버의 최적화 방법은 완전히 달라진다.

### SELECT 절 서브 쿼리

SELECT 절에 사용된 서브쿼리는 내부적으로 임시 테이블을 만들거나 쿼리를 비효율적으로 실행하게 만들지는 않는다.
그래서 서브쿼리가 적절히 인덱스를 사용할 수 있다면 크게 주의할 사항은 없다.
  
일반적으로 SELECT 절에 서브쿼리를 사용하면 그 서브쿼리는 항상 칼럼과 레코드가 하나인 결과를 반환해야 한다.
그 값이 NULL이든 아니든 레코드가 1건이 존재해야 한다는 것인데 MySQL에서는 이 체크 조건이 조금 느슨하다.

```SQL
mysql> SELECT emp_no, (SELECT dept_name FROM departments WHERE dept_name='Sales1')
    -> FROM dept_emp LIMIT 10;
+--------+--------------------------------------------------------------+
| emp_no | (SELECT dept_name FROM departments WHERE dept_name='Sales1') |
+--------+--------------------------------------------------------------+
| 110022 | NULL                                                         |
| 110085 | NULL                                                         |
| 110183 | NULL                                                         |
| 110303 | NULL                                                         |
| 110511 | NULL                                                         |
| 110725 | NULL                                                         |
| 111035 | NULL                                                         |
| 111400 | NULL                                                         |
| 111692 | NULL                                                         |
| 110114 | NULL                                                         |
+--------+--------------------------------------------------------------+

mysql> SELECT emp_no, (SELECT dept_name FROM departments)
    -> FROM dept_emp LIMIT 10;
ERROR 1242 (21000): Subquery returns more than 1 row

mysql> SELECT emp_no, (SELECT dept_no, dept_name FROM departments WHERE dept_name='Sales1')
    -> FROM dept_emp LIMIT 10;
ERROR 1241 (21000): Operand should contain 1 column(s)
```

- 첫 번째 쿼리에서 사용된 서브쿼리는 항상 결과가 0건이다. 하지만 첫 번째 쿼리는 에러를 발생시키지 않고 서브쿼리 결과가 NULL로 채워져서 반환된다.
- 두 번째 쿼리에서 서브쿼리가 2건 이상의 레코드르 반환하는 경우에는 에러가 나면서 쿼리가 종료된다.
- 세 번째 쿼리와 같이 SELECT 절에 사용된 서브쿼리가 2개 이상의 칼럼을 가져오려고 할 때도 에러가 발생한다.

즉 SELECT 절의 서브쿼리에는 `로우 서브쿼리`를 사용할 수 없고, 오로지 `스칼라 서브쿼리`만 사용할 수 있다.

> 서브쿼리는 만들어 내는 결과에 따라 스칼라 서브쿼리(Scalar subquery), 로우 서브쿼리(Row subquery)로 구분할 수 있다.
> 스칼라 서브쿼리는 레코드의 칼럼이 각각 하나인 결과를 만들어내는 서브쿼리며,
> 스칼라 서브쿼리보다 레코드 건수가 많거나 칼럼 수가 많은 결과를 만들어내는 서브쿼리를 로우 서브쿼리라고 한다.

가끔 조인으로 처리해도 되는 쿼리를 SELECT 서브 쿼리를 사용해서 작성할 때도 있는데, 서브쿼리로 실행될 때보다 조인으로 처리할 때가 조금 더 빠르다.
그래서 가능하면 조인으로 쿼리를 작성하는 것이 좋다.

### FROM 절 서브 쿼리

이전 버전의 MySQL 서버에서는 FROM 절에 서브쿼리가 사용되면 항상 임시 테이블을 생성하는 방식으로 처리했다.
하지만 5.7버전부터는 옵티마이저가 FROM 절의 서브쿼리를 외부 쿼리로 병합하는 최적화를 수행하도록 개선됐다.
  
모두 다 가능한 것은 아니고 대표적으로 서브쿼리에 아래 기능들이 사용되면 FROM 절의 서브쿼리는 외부 쿼리로 병합할 수 없다.

- 집합 함수 사용
- DISTINCT
- GROUP BY, HAVING
- LIMIT
- UNION
- SELECT 절에 서브쿼리가 사용된 경우
- 사용자 변수 사용

외부 쿼리와 병합하는 FROM 절의 서브쿼리가 ORDER BY 절을 가진 경우, 외부 쿼리가 GROUP BY나 DISTINCT 같은 기능을 사용하지 않는다면 서브쿼리의 정렬 조건을 외부쿼리로 같이 병합한다. 외부 쿼리에서 GROUP BY나 DISTINCT 같은 기능이 사용되고 있다면 서브쿼리의 정렬 작업은 무의미하기 때문에 서브쿼리의 ORDER BY 절은 무시된다.

### WHERE 절 서브 쿼리

WHERE 절 서브쿼리는 다양한 형태로 사용될 수 있다.

- 동등 또는 크다 작다 비교
- IN 비교
- NOT IN 비교

#### 동등 또는 크다 작다 비교

```SQL
SELECT * FROM dept_emp de
WHERE de.emp_no=(SELECT e.emp_no
                 FROM employees e
                 WHERE e.first_name='Georgi' AND e.last_name='Facello' LIMIT 1);
```

5.5 이전 버전까지는 서브 쿼리의 외부 조건으로 쿼리를 실행하고, 최종적으로 서브쿼리를 체크 조건으로 사용했다.
하지만 이러한 처리 방식은 풀 테이블 스캔이 필요한 경우가 많아 성능 저하가 심각했다.
  
5.5 버전 부터는 서브 쿼리를 먼저 실행한 후 상수로 변환한다. 그리고 상숫값으로 서브쿼리를 대체해서 나머지 쿼리 부분을 처리한다.
  
```sql
mysql> EXPLAIN
-> SELECT *
-> FROM dept_emp de WHERE (emp_no, from_date) = (
->     SELECT emp_no, from_date
->     FROM salaries
->     WHERE emp_no=100001 limit 1);
+----+-------------+----------+------+---------+--------+-------------+
| id | select_type | table    | type | key     | rows   | Extra       |
+----+-------------+----------+------+---------+--------+-------------+
|  1 | PRIMARY     | de       | ALL  | NULL    | 331143 | Using where |
|  2 | SUBQUERY    | salaries | ref  | PRIMARY |      4 | Using index |
+----+-------------+----------+------+---------+--------+-------------+
```

그런데 위와 같이 단일 값 비교가 아닌 튜플 비교 방식이 사용되면 서브쿼리가 먼저 처리되어 상수화되기는 하지만 외부 쿼리는
인덱스를 사용하지 못하고 풀 테이블 스캔을 실행하게된다.

#### IN 비교

```sql
SELECT *
FROM employees e
WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1999-01-01');
```

실제 조인은 아니지만 테이블의 레코드가 다른 테이블의 레코드를 이용한 표현식과 일치하는지를 체크하는 형태를 `세미 조인`이라고 한다.
즉 WHERE 절에 사용된 IN (subquery) 형태의 조건을 조인의 한 방식인 세미 조인이라고 보는 것이다.
  
5.5 버전까지는 세미 조인 최적화가 부족해서 대부분 풀 테이블 스캔을 실행했다. 하지만 5.6 버전부터 세미 조인의 최적화가 많이 개선되었다. MySQL의 세미 조인 최적화는 쿼리 특성이나 조인 관계에 맞게 5개의 최적화 전략을 선택적으로 사용한다.

- 테이블 풀 아웃(Table Pull-out)
- 퍼스트 매치(Firstmatch)
- 루스 스캔(Loosescan)
- 구체화(Materialization)
- 중복 제거(Duplicated Weed-out)

MySQL 8.0을 사용한다면 세미 조인 최적화에 익숙해져야 한다. 

#### NOT IN 비교

IN (subquery)와 비슷한 형태지만 이 경우를 `안티 세미 조인`이라고 부른다.

### CTE(Common Table Expression)

CTE는 이름을 가지는 임시테이블로서, SQL 문장 내에서 한 번 이상 사용될 수 있으며 SQL 문장이 종료되면 자동으로 CTE 임시 테이블은 삭제된다. CTE는 재귀적 반복 실행 여부를 기준으로 Non-recursive와 Recursive CTE로 구분된다.

#### 비 재귀적 CTE

```sql
WITH cte1 AS (SELECT * FROM departments)
SELECT * FROM cte1;
```

CTE 쿼리는 WITH 절로 정의하고 CTE 쿼리로 생성되는 임시 테이블의 이름은 WITH 바로 뒤에 cte1로 정의했다.
cte1 임시 테이블은 한 번만 사용되기 때문에 다음 쿼리처럼 FROM 절의 서브쿼리로 바꿔 사용할 수 있다.
실제 두 쿼리는 실행 계획까지 동일하게 사용한다.

```sql
SELECT *
FROM (SELECT * FROM departments) cte1;
```

```sql
WITH cte1 AS (SELECT * FROM departments),
     cte2 AS (SELECT * FROM dept_emp)
SELECT *
FROM temp1
   INNER JOIN cte2 ON cte2.dept_no=cte1.dept_no;
```

### 윈도우 함수(Window Fuction)

윈도우 함수는 조회하는 현재 레코드를 기준으로 연관된 레코드 집합의 연산을 수행한다.
GROUP BY 또는 집계 함수를 이용하면 다른 레코드의 칼럼값을 참조할 수 있다. 하지만 GROUP BY나 집계 함수를 사용하면 결과 집합의 모양이 바뀐다. 그에 반해, 윈도우 함수는 결과 집합을 그대로 유지하면서 하나의 레코드 연산에 다른 레코드의 칼럼 값을 참조할 수 있다.
    
윈도우 함수를 사용하는 쿼리의 결과에 보여지는 레코드는 FROM 절과 WHERE 절, GROUP BY와 HAVAING에 의해 결정되고
그 이후에 윈도우 함수가 실행된다. 그리고 마지막으로 SELECT 절과 ORDER BY절, LIMIT 절이 실행되어 최종 결과가 반환된다.
  
예를 들어 윈도우 함수를 GROUP BY 칼럼으로 사용하거나, WHERE 절에는 사용할 수 없다. 윈도우 함수 이전에 WHERE, FROM, GROUP BY, HAVING절이 실행되어야 하기 때문이다. 또한 반면 LIMIT을 먼저 실행한 다음 윈도우 함수를 적용할 수 없다.
  
```SQL
AGGREGATE_FUNC() OVER(<partition> <order>) AS window_func_column
```

윈도우 함수는 용도별로 다양한 함수를 사용할 수 있고, 함수 뒤에 OVER 절을 이용해 연산 대상을 파티션하기 위한 옵션을 명시할 수 있다. 이렇게 OVER 절에 의해 만들어진 그룹을 파티션 또는 윈도우라고 한다.

# 

MySQL 서버의 윈도우 함수에는 집계 함수와 비 집계 함수를 모두 사용할 수 있다.
집계 함수는 GROUP BY 절과 함께 사용할 수 있는 함수들을 의미하는데, 집계 함수는 OVER() 절 없이 단독으로도 사용될 수 있고
OVER() 절을 가진 윈도우 함수로도 사용될 수 있다. 반면 비 집계 함수는 반드시 OVER()절을 가지고 있어야 하며 윈도우 함수로만 사용될 수 있다.

MySQL 서버의 윈도우 함수는 8.0 버전에 처음 도입됐으며, 아직 인덱스를 이용한 최적화가 부족한 부분도 있다.

```SQL
SELECT MAX(from_date) OVER(PARTITION BY emp_no) AS max_from_date FROM salaries)

SELECT MAX(from_date) FROM salaries GROUP BY emp_no;
```

윈도우 함수를 사용하는 쿼리는 인덱스를 풀 스캔하고, 레코드 정렬 작업까지 실행했다.  
반면 GROUP BY 절을 사용하는 쿼리는 별도의 정렬 작업 없이 루스 인덱스 스캔으로 MAX(from_date)값을 찾아냈다.
  
윈도우 함수는 salaries 테이블의 모든 레코드 건수만큼의 결과를 만들어야 하지만 GROUP BY절을 사용하는 쿼리는
유니크한 emp_no별로 레코드를 1건씩만 결과를 만들면 된다. 그래서 기본적으로 레코드 건수에서 차이가 날 수 있다.
윈도우 함수를 사용한 쿼리는 예상보다 훨씬 많은 레코드를 가공하고, 그로 인해 MySQL 서버 내부적으로 레코드의 읽고 쓰기가 상당히 많이 발생한다. 
  
가능하다면 윈도우 함수에 너무 의존하지 않는 것이 좋다. 배치 프로그램이라면 윈도우 함수를 사용해도 무방하겠지만 OLTP 에서는 많은 레코드에 대해 윈도우 함수를 적용하는 것은 가능하면 피하는 것이 좋다. 다만 소량의 레코드에 대해선 윈도우 함수를 사용해도 메모리에서 빠르게 처리될 것이므로 특별히 성능에 대해 고민하지 않아도 된다.

### 잠금을 사용하는 SELECT

InnoDB 테이블에 대해서는 레코드를 SELECT할 때 레코드에 아무런 잠금도 걸지 않는데, 이를 잠금 없는 읽기(Non Locking Consistent Read)라고 한다. 
  
하지만 SELECT 쿼리를 이용해 읽은 레코드의 칼럼 값을 애플리케이션에서 가공해서 다시 업데이트하고자 할 때는 SELECT가 실행된 후 다른 트랜잭션이 그 칼럼의 값을 변경하지 못하게 해야 한다. 이럴 때는 레코드를 읽으면서 강제로 잠금을 걸어 둘 필요가 있는데, 이때 사용하는 옵션이 FOR SHARE와 FOR UPDATE 절이다. FOR SHARE는 SELECT 쿼리로 읽은 레코드 에 대해서 읽기 잠금을 걸고, FOR UPDATE는 SELECT 쿼리가 읽은 레코드에 대해서 쓰기 잠금을 건다.


- FOR SHARE 절은 SELECT된 레코드에 대해 읽기 잠금(공유 잠금, Shared lock)을 설정하고 다른 세션에서 해당 레코드를 변경하지 못하게 한다. 물론 다른 세션에서 잠금이 걸린 레코드를 읽는 것은 가능하다.
- FOR UPDATE 절은 쓰기 잠금(배타 잠금, Exclusive lock)을 설정하고, 다른 트랜잭션에서는 그 레코드를 변경하는 것뿐만 아니라 읽기(FOR SHARE 절을 사용하는 SELECT 쿼리)도 수행할 수 없다.

```SQL
SELECT*
FROM employees e
INNER JOIN dept_emp de ON de.emp_no=e. emp_no
INNER JOIN departments d ON d. dept_no=de.dept_no
FOR UPDATE;
```

위 쿼리는 employees 테이블과 dept_emp 테이블, departments 테이블을 조인해서 읽으면서 FOR UPDATE 절을 사용한다. 그래서 InnoDB 스토리지 엔진은 3개 테이블에서 읽은 레코드에 대해 모두 쓰기 잠금을 걸게 된다. 
  
8.0 버전부터는 다음과 같이 잠금을 걸 테이블을 선택할 수 있도록 기능이 개선됐다. 다음 예제와 같이 FOR UPDATE 뒤에 "OF 테이블" 절을 추가 하면 해당 테이블에 대해서만 잠금을 걸게 된다. 테이블에 대해 별칭이 사용된 경우에는 별칭을 명시해야 한다. SELECT 쿼리에 사용된 테이블 중에서 특정 테이블만 잠금을 획득하는 옵션은 FOR UPDATE
와 FOR SHARE 절 모두 적용할 수 있다.

```SQL
SELECT *
FROM employees e
INNER JOIN dept_emp de ON de. emp_no=e. emp_no
INNER JOIN departments d ON d.dept_no=de.dept_no
WHERE e. emp_no=10001
FOR UPDATE OF e;
```

#### NOWAIT & SKIP LOCKED

8.0 버전부터는 NOWAIT과 SKIP LOCKED 옵션을 사용할 수 있게 기능이 추가됐다. 지금까지의 MySQL 잠금은 누군가가 레코드를 잠그고 있다면 다른 트랜잭션은 그 잠금이 해제될 때까지 기다려야 했다. 때로는 일정 시간이 지나면 잠금 획득 실패 에러 메시지를 받을 수도 있었다. 하지만 이런 작동 방식은 휴대폰의 화면을 보면서 응답을 기다리고 있을 사용자를 생각하면 때로는 적절한 작동 방식이 아닐 수도 있다. 
  
애플리케이션의 어떤 기능에서는 레코드가 이미 잠겨진 상태라면 그냥 무시하고 즉시 에러를 반환하면 응용 프로그램에서 다른 처리를 수행하거나 다시 트랜잭션을 시작하도록 구현해야할 때도 있다. 이럴 때 SELECT 쿼리의 마지막에 NOMAIT 옵션을 사용하면 된다. FOR UPDATEL FOR SHARE 절이 없는 SELECT 쿼리는 잠금 대기 자체가 없기 때문에 NOWAIT 옵션을 사용하는 것은 의미가 없다
  
```SQL
SELECT * FROM employees
WHERE emp_no=10001
FOR UPDATE NOWAIT;

ERROR 3572 (HY000): Statement aborted
because lock(5) could not be acquired immediately and NOMAIT is set.
```

NOMAIT 옵션을 사용하면 SELECT 쿼리가 해당 레코드에 대해 즉시 잠금을 획득했다면 NOWAIT 옵션이 없을 때와 동일하게 실행된다. 하지만 해당 레코드가 다른 트랜잭션에 의해서 잠겨진 상태라면 다음과 같이 에러를 반환하면서 쿼리는 즉시 종료된다.

```SQL
SELECT * FROM salaries WHERE emp_no=10001 FOR UPDATE SKIP LOCKED;
```

SKIP LOCKED 옵션은 SELECT하려는 레코드가 다른 트랜잭션에 의해 이미 잠겨진 상태라면 에러를 반환하지 않는다. 잠긴 레코드는 무시하고 잠금이 걸리지 않은 레코드만 가져온다.


## Reference 

**위 글은 책 RealMySQL 8.0 2권을 구입하여 읽고 정리한 내용입니다.**
- [도서 홈페이지 https://wikibook.co.kr/realmysql802/](https://wikibook.co.kr/realmysql802/)
