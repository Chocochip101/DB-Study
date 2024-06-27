DBMS는 많은 데이터를 안전하게 저장 및 관리하고 사용자가 원하는 데이터를 빠르게 조회할 수 있게 해주는 것이 목적이다. 이러한 목적을 달성하려면 옵티마이저가 사용자의 쿼리를 최적으로 처리될 수 있게 하는 **쿼리의 실행 계획을 수립**할 수 있어야 한다. 하지만 옵티마이저가 관리자나 사용자의 개입 없이 항상 좋은 실행 계획을 만들어낼 수 있는 것은 아니다. 

DBMS 서버는 이러한 문제점을 관리자나 사용자가 보완할 수 있도록 **EXPLAIN 명령**으로 옵티마이저가 수립한 실행 계획을 확인할 수 있게 해준다. 하지만 M**ySQL 서버에서 보여주는 실행 계획을 읽고 이해하려면 MySQL 서버가 데이터를 처리하는 로직을 이해할 필요가 있다.** 그런데 이것이 처음 MySQL 서버를 접하는 사용자에게는 그렇게 쉬운 일은 아니다.

# 통계 정보 
MySQL 서버는 5.7 버전까지 테이블과 인덱스에 대한 개괄적인 정보를 가지고 실행 계획을 수립했다. 하지만 이는 테이블 칼럼의 값들이 실제 어떻게 분포돼 있는지에 대한 정보가 없기 때문에 실행 계획의 정확도가 떨어지는 경우가 많았다. 그래서 MySQL 8.0 버전부터는 **인덱스되지 않은 칼럼들에 대해서도 데이터 분포도를 수집해서 저장하는 히스토그램(Histogram) 정보가 도입됐다**. 히스토그램이 도입됐다고 해서 기존의 테이블이나 인덱스의 통계 정보가 필요치 않은 것은 아니다. 여기서는 테이블 및 인덱스에 대한 통계 정보와 히스토그램을 나누어 살펴보겠다. 

## 테이블 및 인덱스 통계 정보 
비용 기반 최적화에서 가장 중요한 것은 통계 정보다. 통계 정보가 정확하지 않다면 전혀 엉뚱한 방향으로 쿼리를 실행할 수 있기 때문이다. 

> 예를 들어, 1억 건의 레코드가 저장된 테이블의 통계 정보가 갱신되지 않아서 레코드가 10건 미만인 것처럼 돼 있다면 옵티마이저는 실제 쿼리를 실행할 때 인덱스 레인지 스캔이 아니라 테이블을 처음부터 끝까지 읽는 방식(풀 테이블 스캔)으로 실행해 버릴 수도 있다. 부정확한 통계 정보 탓에 0.1초에 끝날 쿼리에 1시간이 소요될 수도 있다.

### MySQL 서버의 통계 정보
MySQL 5.5 버전까지는 테이블의 통계 정보가 메모리에 저장되어 관리되었기에, 서버가 재시작되면 지금까지 수집된 정보가 모두 사라진다. MySQL 버전 5.6 버전부터는 각 테이블의 통계정보를 mysql 데이터베이스의 `innodb_index_stats` 테이블과 `innodb_table_stats` 테이블로 관리할 수 있게 개선되었다.

#### 통계 정보 영구 저장
MySQL 5.6에서 테이블을 생성할 때는 `STATS_PERSISTENCE` 옵션을 설정할 수 있는, 이 설정값에 따라 테이블 단위로 영구적인 통계 정보를 보관할지 결정할 수 있다.

- `STATS_PERSISTENCE`=0: 테이블의 통계 정보를 MySQL 5.5 이전의 방식대로 관리하며, mysql 데이터베이스의 `innodb_index_stats`와 `innodb_table_stats` 테이블에 저정하지 않는다.
- `STATS_PERSISTENCE`=1: 테이블의 통계 정보를 mysql 데이터베이스의 `innodb_index_stats`와 `innodb_table_stats` 테이블에 저정한다.
- `STATS_PERSISTENCE`=DEFAULT: 테이블을 생성할때 별도로 `STATS_PERSISTENCE` 옵션을 설정하지 않은 것과 동일하며, 테이블의 통계를 영구적으로 관리할지 말지를 `innodb_stats_persistence` 시스템 변수의 값으로 결정한다.

#### 통계 수집 정보 변경
**자주 통계 정보가 갱신되면 의도치 않게 응용 프로그램의 쿼리를 풀 테이블 스캔으로 실행될 수 있다.** 이러한 문제를 `innodb_stats_auto_recalc` 시스템 변수의 값을 OFF로 설정해서 **통계 정보가 자동을 갱신되는 것을 막을 수 있다.** 통계 정보의 자동 수집을 `STATS_AUTO_RECALC` 옵션을 통해 테이블 단위로 조정할 수 있다.

#### 샘플링 블록 단위

- `innodb_stats_transient_sample_pages`: 자동으로 통계 정보 수집이 실행될 때 **정해진 시스템 변수 값만큼의 페이지만 임의로 샘플링해서 분석**하고 그 결과를 통계 정보로 활용한다.
- `innodb_stats_persistence_sample_pages`: ANALYZE TABLE 명령이 실행되면 임의로 시스템 변수 값의 페이지만 샘플링해서 분석하고 그 결과를 영구적인 통계 정보 테이블에 활용한다.

`innodb_stats_persistence_sample_pages`를 높게하여 정확한 통계 정보를 수집하고, 쿼리의 성능을 높일 수 있다. 하지만 통계 정보 수집 시간이 길어지고, 많은 시간이 소요될 수 있다.
## 히스토그램
MySQL 8.0 버전부터 **칼럼의 데이터 분포도**를 참조할 수 있는 히스토그램 정보를 활용할 수 있게 되었다.

### 히스토그램 정보 수집 및 삭제
MySQL 8.0 버전에서 히스토그램 정보는 칼럼 단위로 관리되는데, 이는 자동으로 수집되지 않고 `ANALYZE TABLE ... UPDATE HISTOGRAM` 명령을 실행해 수동으로 수집 관리된다. 수집된 히스토그램 정보는 시스템 딕셔너리에 함께 저장되고, MySQL 서버가 시작될 때 딕셔너리의 히스토그램 정보를 `information_schema` 데이터베이스의 `column_statistics` 테이블로 로드한다. 그래서 실제 히스토그램 정보를 조회하려면 `column_statistics` 테이블을 SELECT해서 참조할 수 잇다.

#### 히스토그램 타입
MySQL 8.0 버전에서는 두 가지 히스토그램 종류가 존재한다.

- Singleton(싱글톤 히스토그램): **칼럼값 개별로 레코드 건수를 관리**하는 히스토그램으로, Value-Based 히스토그램 또는 도수 분포라고 불린다.
- Equi-Height(높이 균등 히스토그램): **칼럼값의 범위를 고려한 균등한 개수로 구분해서 관리**하는 히스토그램으로, Height-Balanced 히스토그램이라고도 불린다.

히스토그램은 버킷 단위로 구분되어 레코드 건수나 칼럼값의 범위가 관리되는데, 싱글톤 히스토그램은 칼럼이 가지는 값별로 버킷이 할당되며 높이 균형 히스토그램에서는 개수가 균등한 칼럼값의 범위별로 하나의 버킷이 할당된다. 싱글톤 히스토그램은 각 버킷이 칼럼의 값과 발생 빈도의 비율의 2개 값을 가진다. 반면 높이 균형 히스토그램은 각 버킷이 범위 시작 값과 마지막 값, 그리고 발생 빈도율과 버킷에 포함된 유니크한 값의 개수 등 4개의 값을 가진다.

#### 삭제
히스토그램의 삭제 작업은 테이블의 데이터를 참조하는 것이 아니라, 딕셔너리의 내용만 삭제하기 때문에 다른 쿼리 처리의 성능에 영향을 주지 않고 즉시 완료된다. 하지만 히스토그램이 사라지면 쿼리의 실행 계획이 달라질 수 있으니 유의해야 한다.

### 히스토그램의 용도

MySQL 서버에 히스토그램이 도입되기 이전에도 테이블과 인덱스에 대한 정보는 있었으나, 서버가 가지고 있던 통계 정보는 테이블의 전체 레코드 건수와 인덱스된 칼럼이 가지는 유니크 값의 개수 정도였다. 예를 들어, 테이블의 레코드가 1000건이고 어떤 칼럼의 유니크한 값 개수가 100개였다면 MySQL 서버는 대략 10개의 레코드가 일치할 것이라고 예측한다. 하지만 실제 응용 프로그램의 데이터는 균등한 분포를 가지고 있지 않다. 이러한 단점을 보완하기 위해 히스토그램은 특정 칼럼이 가지는 모든 값에 대한 분포도 정보를 가지지는 않지만 각 범위별로 레코드의 건수와 유니크한 값의 개수 정보를 가지기 때문에 훨씬 정확히 예측할 수 있다. 이를 통해 어느 테이블을 먼저 읽어야 조인의 횟수를 줄일 수 있을지 옵티마이저가 더 정확히 파악할 수 있다.

### 히스토그램의 인덱스

## 코스트 모델
MySQL 쿼리를 처리하려면 다음과 같은 다양한 작업을 필요로 한다.

- 디스크로부터 데이터 페이지 읽기
- 메모리(InnoDB 버퍼 룰)로부터 데이터 페이지 읽기
- 인덱스 키 비교
- 레코드 평가
- 메모리 임시 테이블 작업
- 디스크 임시 테이블 작업

MySQL 서버는 사용자의 쿼리에 대한 이러한 다양한 작업이 얼마나 필요한지 예측하고 전체 작업 비용을 계산한 결과를 바탕으로 최적의 실행 계획을 찾는다. 이렇게 **전체 쿼리의 비용을 계산하는데 필요한 단위 작업들의 비용을 코스트 모델**이라고 한다. MySQL 5.7 이전 버전까지는 이런 작업들의 비용을 MySQL 서버 소스 코드에 상수화해서 사용했다. 하지만 이 작업들의 비용은 MySQL 서버가 사용하는 하드웨어에 따라 달라질 수 있기 때문에 예전 버전처럼 고정된 비율을 일률적으로 적용하는 것은 최적의 실행 계획 수립에 있어서 방해 요소였다.

이런 단점을 보완하기 위해 MySQL 5.7 버전부터 MySQL 서버의 소스 코드에 상수화돼 있던 각 단위 작업의 비용을 DBMS 관리자가 조정할 수 있게 개선됐다. 하지만 MySQL 5.7 버전에서는 인덱스되지 않은 컬럼의 데이터 분포(히스토그램)나 메모리에 상주 중인 페이지의 비율 등 비용 계산과 연관된 부분의 정보가 부족한 상태였다. MySQL 8.0 버전으로 업그레이드되면서 비로소 **컬럼의 데이터 분포를 위한 히스토그램과 각 인덱스별 메모리에 적재된 페이지의 비율이 관리되고 옵티마이저의 실행 계획 수립에 사용되기 시작했다.**

MySQL 8.0 서버의 코스트 모델은 다음 2개 테이블에 저장돼 있는 설정값을 사용하는데 두 테이블 모두 mysql DB에 존재한다.

- server_cost : 인덱스를 찾고 레코드를 비교하고 임시 테이블 처리에 대한 비용 관리
- engine_cost : 레코드를 가진 데이터 페이지를 가져오는데 필요한 비용 관리

각 테이블은 공통으로 다음 5개 컬럼을 가지고 있다.

|컬럼 명|설명|
|---|---|
|cost_name	|코스트 모델의 각 단위 작업|
|default_value	|각 단위 작업의 비용|
|cost_value	|DBMS 관리자가 설정한 값|
|last_updated	|단위 작업의 비용이 변경된 시점|
|comment	|비용에 대한 추가 설명|

`engine_cost` 테이블은 추가로 다음 2개의 컬럼을 더 가지고 있다.

|컬럼 명	|설명|
|---|---|
|engine_name|비용이 적용된 스토리지 엔진|
|device_type|디스크 타입|

MySQL 8.0 버전의 코스트 모델에서는 지원하는 단위 작업은 다음과 같이 8개다.

||cost_name|default_value|설명|
|---|---|---|---|
|engine_cost|io_block_read_cost|1.00|디스크 데이터 페이지 읽기|
|engine_cost|memory_block_read_cost|0.25|메모리 데이터 페이지 읽기|
|server_cost|disk_template_create_cost|20.00|디스크 임시 테이블 생성|
|server_cost|disk_template_row_cost|0.5|디스크 임시 테이블의 레코드 읽기|
|server_cost|key_compare_cost|0.05|인덱스 키 비교|
|server_cost|memory_temptable_create_cost|1.00|메모리 임시 테이블 생성|
|server_cost|memory_temptable_row_cost|0.1|메모리 임시 테이블의 레코드 읽기|
|server_cost|row_evaluate_cost|0.1|레코드 비교|

`row_evaluate_cost`는 스토리지 엔진이 반환한 레코드가 쿼리의 조건에 일치하는지 평가하는 단위 작업을 의미하는데 `row_evaluate_cost` 값이 증가할수록 풀 테이블 스캔과 같이 많은 레코드를 처리하는 쿼리의 비용이 높아지고 반대로 레인지 스캔과 같이 상대적으로 적은 수의 레코드를 처리하는 쿼리의 비용이 낮아진다.

`key_compare_cost`는 키 값의 비교 작업에 필요한 비용을 의미하는데 증가할수록 레코드 정렬과 같이 키 값 비교 처리가 많은 경우 쿼리의 비용이 높아진다.

모든 정보가 사용자에게 표시되는 것은 아니기 때문에 작업의 비용을 직접 계산하는 것은 상당히 어렵다.

**코스트 모델에서 중요한 것은 각 단위 작업에 설정되는 비용 값이 커지면 어떤 실행 계획들이 고비용으로 바뀌고 어떤 실행 계획들이 저비용으로 바뀌는지 파악하는 것이다.**

대표적으로 각 단위 작업의 비용이 변경되면 예상할 수 있는 결과들은 다음과 같다.

- `key_compare_cost` 비용을 높이면 MySQL 서버 옵티마이저가 가능하면 정렬을 수행하지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.
- `row_evaluate_cost` 비용을 높이면 풀 스캔을 실행하는 쿼리들의 비용이 높아지고 MySQL 옵티마이저는 가능하면 인덱스 레인지 스캔을 사용하는 실행 계획을 선택할 가능성이 높아진다.
- `disk_temptable_create_cost`와 `disk_temptable_row_cost` 비용을 높이면 옵티마이저는 디스크에 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아진다.
- `memory_temptable_create_cost` 와` memory_temptable_row_cost` 비용을 높이면 MySQL 서버 옵티마이저는 메모리 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아진다
- `io_block_read_cost` 비용이 높아지면 MySQL 서버 옵티마이저는 가능하면 InnoDB 버퍼 풀에 데이터 페이지가 많이 적재돼 있는 인덱스를 사용하는 실행 계획을 선택할 가능성이 높아진다.
- `memory_block_read_cost` 비용이 높아지면 MySQL 서버는 InnoDB 버퍼 풀에 적재된 데이터 페이지가 상대적으로 적다고 하더라도 그 인덱스를 사용할 가능성이 높아진다.

각 단위 작업의 비용을 사용자가 변경할 수 있는 기능을 제공한다고 해서 이 비용들을 꼭 바꿔서 사용해야 하는 것은 아니다. 코스트 모델은 MySQL 서버가 사용하는 하드웨어와 MySQL 서버 내부적인 처리 방식에 대한 깊이있는 지식을 필요로 한다. MySQL 서버에 적용된 기본값으로도 MySQL 서버는 20년이 넘는 시간 동안 수많은 응용프로그램에서 잘 사용돼 왔다.

# 실행 계획 확인
MySQL 서버의 실행 계획은 DESC 또는 EXPLAIN 명령으로 확인할 수 있다. 그리고 MySQL 8.0 버전부터 EXPLAIN 명령에 사용할 수 있는 새로운 옵션이 추가되었다.

## 실행 계획 출력 포맷
MySQL 8.0 버전부터는 `EXPLAIN EXTENDED`과 `EXPLAIN PARTITIONS` 명령 내용이 통합되어 PARTITIONS나 EXTENDED 옵션은 문법에서 제거됐다.

## 쿼리의 실행 시간 확인
- MySQL 8.0.18 버전부터는 쿼리의 실행 계획과 단계별 소요된 시간 정보를 확인할 수 있는 EXPLAIN ANALYZE 기능이 추가됐다.
- EXPLAIN ANALYZE 명령은 항상 결과를 TREE 포맷으로 출력한다.

```sql
mysql> EXPLAIN ANALYZE
SELECT e, emp_no, avg(s. salary)
FROM employees e
INNER JOIN salaries s ON S,emp_no=e.emp_no
AND s. salary)50000
AND s. from_date{=' 1990-01-01'
AND s. to_date) '1990-01-01'
WHERE e. first_name= 'Matt'
GROUP BY e. hire_date \G

A) -> Table scan on <temporary> (actual time 0,001.0.004 rows=48 loops=1)

B) 	-> Aggregate using temporary table (actual time=3.799..3,808 rows=48 loops=1)

C)		-> Nested loop inner join (cost=685.24 rows=135)
(actual time=0.367..3,602 rows=48 loops=1)

D)			-> Index lookup on e using ix_firstname (first_name='Matt*) (cost=215.08 rows=233)
(actual time=0,348.1,046 rows=233 loops=1)

E)				-> Filter: ((s.salary > 50000) and (s, from_date <= DATE ' 1990-01-01')
and (s.to_date › DATE'1990-01-01')) (cost=0.98 rons=1)
(actual time=0.009.0.011 rows=0 loops=233)

F)					-> Index lookup on s using PRIMARY (emp_noze.emp_no) (cost=0,98 rows=18)
(actual time=0.007.0.009 rows=10 loops=233)
```
실행 계획의 A) ~ F) 번호는 설명의 편의를 위해서 임의로 추가한 것이다.
TREE 포맷의 실행 계획에서 들여쓰기는 호출 순서를 의미하며 실제 실행 순서는 다음 기준으로 읽으면 된다.

- 들여쓰기가 같은 레벨에서는 상단에 위치한 라인이 먼저 실행
- 들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행

위 쿼리의 실행 계획은 다음의 실행 순서를 의미한다.

1. D) Index lookup on e using ix_firstname
2. F) Index lookup on s using PRIMARY
3. E) Filter
4. C) Nested loop inner join
5. B) Aggregate using temporary table
6. A) Table scan on < temporary >

한글로 실행 계획을 풀어 쓸 수 있다.

1. employees 테이블의 1X_firstname 인덱스를 통해 first name='watt' 조건에 일치하는 레코드를 찾고
2. salaries 테이블의 PRIMARY 키를 통해 emp.nO가 (1)번 결과의 emp_no와 동일한 레코드를 찾아서
3. ((s. salary > 50000) and (s. from date <= DATE' 1990-01-01) and (s.to date › DATE' 1990-01-01))
조건에 일치하는 건만 가져와
4. 1번과 3번의 결과를 조인해서
5. 임시 테이블에 결과를 저장하면서 GROUP BY 집계를 실행하고
6. 임시 테이블의 결과를 읽어서 결과를 반환한다.


# 실행 계획 분석
실행 계획이 어떤 접근 방법을 사용해서 어떤 최적화를 수행하는지 그리고 어떤 인덱스를 사용하는지 등을 이해하는 것이 중요하다.

```sql
mysql> EXPLAIN
		SELECT *
        FROM employees e
        	INNER JOIN salaries s ON s.emp_no = e.emp_no
        WHERE first_name='ABC'
```



## id 칼럼
하나의 SELECT 문장은 다시 1개 이상의 하위 SELECT 문장을 포함할 수 있다.

```sql
mysql> SELECT ...
		FROM (SELECT ... FROM tb_test1) tb1, tb_test2 tb2
        WHERE tb1.id = tb2.id;
```

위 쿼리 문장에 있는 SELECT 쿼리를 다음과 같이 분리해서 생각해볼 수 있다.

```sql
mysql> SELECT ... FROM tb1_test1;
mysql> SELECT ... FROM tb1, tb_test2 tb2 WHERE tb1.id = tb2.id;
```
- 실행 계획에서 가장 왼쪽에 표시되는 id 컬럼은 단위 SELECT 쿼리별로 부여되는 식별자 값이다.
- 이 예제 쿼리의 경우 실행 계획에서 최소 2개의 id 값이 표시될 것이다.

- 하나의 SELECT 문장 안에서 여러 개의 테이블을 조인하면 조인되는 테이블의 개수만큼 실행 계획 레코드가 출력되지만 같은 id 값이 부여된다.

다음 예제는 SELECT 문장은 하나인데 여러 개의 테이블이 조인되는 경우에는 id 값이 증가하지 않고 같은 id 값이 부여된다.

```sql
mysql> EXPLAIN
		SELECT e.emp_no, e.first_name, s.from_date, s.salary
        FROM employees e, salaries s
        WHERE e.emp_no=s.emp_no LIMIT 10;
```
반대로 다음 쿼리의 실행 계획에서는 쿼리 문장이 3개의 단위 SELECT 쿼리로 구성돼 있으므로 실행 계획의 각 레코드가 각기 다른 id 값을 지닌 것을 확인할 수 있다.
```sql
mysql> EXPLAIN
		SELECT 
        ( (SELECT COUNT(*) FROM employees) + (SELECT COUNT(*) FORM departments) ) AS total_count;
```

여기서 주의해야 할 것은 id 컬럼이 테이블의 접근 순서를 의미하지는 않는다는 것이다.
EXPLAIN FORMAT=TREE 명령으로 확인해보면 순서를 알 수 있다.

## select_type 칼럼
각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 컬럼이다. select_type 컬럼에 표시될 수 있는 값은 다음과 같다.

- SIMPLE: UNION이나 서브쿼리를 사용하지 않는 단순한 SELECT 쿼리
- PRIMARY: UNION이나 서브쿼리를 가지는 SELECT 쿼리의 실행 계획에서 가장 바깥쪽에 있는 단위 쿼리
- UNION: UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리
- DEPENDENT UNION: UNION 이나 UNION ALL로 결합된 쿼리가 외부 쿼리에 의해 영향을 받는 쿼리
- UNION RESULT: UNION 결과를 담아두는 테이블

> MySQL 8.0 이전 버전에서는 UNION ALL이나 UNION 쿼리는 모두 임시 테이블을 생성했는데 MySQL 8.0 버전부터는 UNION ALL의 경우 임시 테이블을 사용하지 않도록 기능이 개선됐다.
따라서 UNION ALL을 하면 실행계획에 보이지 않는다. 하지만 UNION은 여전히 임시 테이블을 사용해 결과를 버퍼링한다. UNION RESULT는 실제 쿼리에서 단위 쿼리가 아니기 때문에 별도의 id 값은 부여되지 않는다.

- SUBQUERY: select_type의 SUBQUERY는 FROM 절 이외에서 사용되는 서브쿼리만을 의미한다.

```sql
mysql> EXPLAIN
		SELECT e.first_name,
        	(SELECT COUNT(*)
            FROM dept_emp de, dept_manager dm
            WHERE dm.dept_no=de.dept_no) AS cnt
        FROM employees e WHERE e.emp_no=10001;
```

- DEPENDENT SUBQUERY: 서브쿼리가 바깥쪽 SELECT 쿼리에서 정의된 칼럼을 사용하는 경우

```sql
mysql> EXPLAIN
		SELECT e.first_name,
        	(SELECT COUNT(*)
            FROM dept_emp de, dept_manager dm
            WHERE dm.dept_no=de.dept_no) AS cnt
        FROM employees e 
        WHERE e.first_name='Matt';
```
그래서 위의 예시를 살펴보면 FROM 절에 사용되는 쿼리는 DERIVED로 표기되고 IN 절에 사용된 쿼리는 DEPENDENT SUBQUERY로 표기된다.


- DERIVED: 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블을 생성하는 것을 의미한다. select_type이 DERIVED인 경우에 생성되는 임시 테이블을 파생 테이블이라고도 한다.
- DEPENDENT DERIVED: 서브 쿼리 중 외부에 영향을 받는 첫 번째 쿼리 MySQL 8.0 버전부터 LATERAL JOIN 기능이 추가되면서 FROM 절의 서브쿼리에서도 외부 컬럼을 참조할 수 있게 됐다.

```sql
mysql> SELECT *
		FROM employees e
        LEFT JOIN LATERAL
        	(SELECT *
            	FROM salaries s
                WHERE s.emp_no=e.emp_no
                ORDER BY s.from_date DESC LIMIT 2) AS s2 ON s2.emp_no=e.emp_no
```

> 래터럴 조인의 경우에는 LATERAL 키워드를 사용해야 하며 LATERAL 키워드가 없는 서브쿼리에서 외부 컬럼을 참조하면 오류가 발생한다.

하나의 쿼리 문장에 서브쿼리가 하나만 있더라도 실제 그 쿼리가 한 번만 실행되는 것은 아니다. 그런데 조건이 똑같은 서브쿼리가 실행될 때는 다시 실행하지 않고 이전의 실행 결과를 그대로 사용할 수 있게 서브쿼리의 결과를 내부적인 캐시 공간에 담아둔다. 여기서 언급하는 서브쿼리 캐시는 쿼리 캐시나 파생 테이블(DERIVED)과는 전혀 무관한 기능이다.

- UNCACHEABLE SUBQUERY: SUBQUERY는 바깥쪽 영향을 받지 않으므로 처음 한 번만 실행해서 그 결과를 캐시하고 필요할 때 캐시 된 결과를 이용한다. DEPENDENT SUBQUERY는 바깥쪽 쿼리 컬럼의 값 단위로 캐시해두고 사용한다.
- UNCACHEABLE UNION:UNCACHEABLE과 UNION이 혼합된 select_type을 의미한다.
- MATERIALIZED: 
## table 칼럼
MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시된다. 테이블의 이름에 별칭이 부여된 경우에는 별칭이 표시된다. 만약 별도의 테이블을 사용하지 않는 SELECT 쿼리인 경우에는 table 컬럼에 NULL이 표시된다. table 컬럼에 "<>"로 둘러싸인 이름이 명시되는 경우 이는 임시 테이블을 의미한다.

## partitions 칼럼
MySQL 8.0 버전부터 EXPLATIN 명령으로 파티션 관련 실행 계획까지 모두 확인할 수 있게 변경됐다. 실행 계획에서 조회 조건에 맞는 파티션만 접근하고 불필요한 나머지 파티션에 대해서는 분석을 실행하지 않는다. 이를 파티션 프루닝이라고 한다.

> 실행 계획을 보면 파티션 특정 파티션만 접근했다는 것을 알 수 있다.
이때 주의해야될 점은 type에 ALL로 표기된다는 점이다.
partition은 물리적으로 개별 테이블처럼 별도의 저장 공간을 가지기 때문에 type에 파티션 일부만 읽는 쿼리라도 테이블 풀 스캔처럼 ALL로 표기된다.

## type 칼럼

type 이후 컬럼은 MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 나타낸다. MySQL의 메뉴얼에서는 type 컬럼을 조인 타입으로 소개한다. 또한 MySQL에서는 하나의 테이블로부터 레코드를 읽는 작업도 조인처럼 처리한다. 그래서 SELECT 쿼리의 테이블 개수에 관계없이 실행 계획의 type 컬럼을 조인 타입이라고 명시하고 있다.

ALL을 제외한 나머지 모두 인덱스를 사용하는 접근 방법이다. 하나의 단위 SELECT 쿼리는 위의 접근 방법 중에서 단 하나만 사용할 수 있다. 또한 index_merge를 제외한 나머지 방법은 하나의 인덱스만 사용한다.

- system: 레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 system 이라고 한다. 이 접근 방법은 InnoDB 스토리지 엔진을 사용하는 테이블에서는 나타나지 않는다.
- const: 테이블의 레코드 건수와 관계없이 쿼리가 프라이머리 키나 유니크 키 칼럼을 이용하는 WHERE 조건절을 가지고 있으며 반드시 1건을 반환하는 쿼리의 처리 방식을 const라고 한다. 다른 DBMS에서는 이를 유니크 인덱스 스캔이라고 표현한다.
- eq_ref: 조인에서 첫 번째 읽은 테이블의 컬럼 값을 이용해 두 번째 테이블을 프라이머리 키나 유니크 키로 동등 조건 검색(두 번쨰 테이블은 반드시 1건의 레코드만 반환)

- ref: 조인의 순서와 인덱스 종류에 관계없이 동등 조건으로 검색 (1건의 레코드만 반환된다는 보장이 없어도 됨)

eq_ref와는 달리 조인의 순서와 관계없이 사용되며 또한 프라이머리키나 유니크키 등의 제약조건도 없다. 인덱스의 종류와 관계없이 동등 조건으로 검색할 때는 ref 접근 방법이 사용된다.

- fulltext: 전문 검색(Full-text Search) 인덱스를 사용해 레코드를 읽는 접근 방법이다.
전문 검색용 인덱스가 테이블에 정의돼 있어야 오류가 발생하지 않고 사용이 가능하다.
전문 검색 인덱스를 사용하는 fulltext보다 일반 인덱스를 이용하는 range 접근 방법 더 빨리 처리되는 경우가 잦아서 조건별로 성능을 확인해봐야 한다.
- ref_or_null: ref 접근 방법과 같은데 NULL 비교가 추가된 형태이다. 접근 방법의 이름 그대로 ref 방식 또는 NULL 비교 (IS NULL) 접근 방법을 의미한다
- unique_subquery: WHERE 조건절에 사용될 수 있는 IN 형태의 쿼리를 위한 접근 방법이다.
서브쿼리에서 중복되지 않는 유니크한 값만 반환할 때 이 접근 방법을 사용한다. IN(subquery) 형태의 조건에서 subquery의 반환 값에 중복이 없으므로 별도의 중복 제거 작업이 필요하지 않음
- index_subquery: IN 연산자의 특성상 IN(subquery) 또는 IN(상수 나열) 형태의 조건은 괄호 안에 있는 값의 목록중에서 중복된 값이 먼저 제거돼야 한다. 이때 서브쿼리 결과의 중복된 값을 인덱스를 이용해서 제거할 수 있을 때 index_subquery 접근 방법이 사용된다. IN(subquery) 형태의 조건에서 subquery의 반환 값에 중복된 값이 있을 수 있지만 인덱스를 이용해 중복된 값을 제거할 수 있음

- range: 레인지 스캔 형태의 접근 방법이다. range는 인덱스를 하나의 값이 아니라 범위로 검색하는 경우를 의미하는데 주로 <, >, IS NULL, BETWEEN, IN, LIKE 등의 연산자를 이용해 인덱스를 검색할 때 사용된다.

- index_merge: 2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들어낸 후 그 결과를 병합해서 처리하는 방식이다. index_merge는 효율이 좋지 않다. 여러 인덱스를 읽어야 하므로 일반적으로 range 접근 방법보다 효율성이 떨어진다. 전문 검색 인덱스를 사용하는 쿼리에서는 index_merge가 적용되지 않는다. index_merge 접근 방법으로 처리된 결과는 항상 2개 이상의 집합이 되기 때문에 그 두 집합의 교집합이나 합집합 또는 중복 제거와 같은 부가적인 작업이 더 필요하다.

```sql
mysql> EXPLAIN
		SELECT * FROM employees
        WHERE emp_no BETWEEN 1001 AND 11000
        	OR first_name='Smith';
```

- index: index 접근 방법은 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔을 의미한다.
index 접근 방법은 다음과 같은 조건을 충족하는 경우 사용된다. 
  - range나 const, ref 같은 접근 방법으로 인덱스를 사용하지 못하는 경우
  - 인덱스에 포함된 컬럼만으로 처리할 수 있는 쿼리인 경우
  - range나 const, ref 같은 접근 방법으로 인덱스를 사용하지 못하는 경우
  - 인덱스를 이용한 정렬이나 그루핑 작업이 가능한 경우
  
- ALL: 우리가 흔히 알고 있는 풀 테이블 스캔을 의미하는 접근 방법이다. 대량의 디스크 I/O를 유발하는 작업을 한꺼번에 많은 페이지를 읽어 들이는 기능인 리드 어헤드를 제공한다. 쿼리를 튜닝한다는 것이 무조건 풀 테이블 스캔, 풀 인덱스 스캔을 사용하지 못하게 하는 것은 아니다. 온라인 트랜잭션 환경(웹 서비스)에서는 적합하지 않다.

## possible_keys 컬럼
MySQL 옵티마이저는 쿼리를 처리하기 위해 여러 가지 처리 방법을 고려하고 그중에서 비용이 가장 낮을 것으로 예상하는 실행 계획을 선택해 쿼리를 실행한다. 그런데 possible_keys 컬럼에 있는 내용은 옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록일 뿐이다. 즉 말 그대로 '사용될법했던 인덱스 목록'인 것이다.
> 이 컬럼은 무시해되 되는데 절대 possible_keys 컬럼에 인덱스 이름이 나열됐다고 해서 그 인덱스를 사용한다고 판단하지 말자.

## key 컬럼
최종 선택된 실행 계획에서 사용하는 인덱스를 의미한다.
PRIMARY인 경우에는 프라이머리 키를 사용한다는 의미이며 그 이외의 값은 모두 테이블이나 인덱스를 생성할 때 부여했던 고유 이름이다.
실행 게획의 type이 ALL일 때와 같이 인덱스를 전혀 사용하지 못하면 key 컬럼은 NULL로 표시된다.

> MySQL 서버에 프라이머리 키는 별도의 이름을 부여할 수 없으며 기본적으로 PRIMARY 라는 이름을 가진 그 밖의 나머지 인덱스는 모두 테이블을 생성하거나 인덱스를 생성할 때 이름을 부여할 수 있다. 실행 계획뿐만 아니라 쿼리의 힌트를 사용할 때도 프라이머리 키를 지칭하고 싶다면 "PRIMARY"라는 키워드를 사용하면 된다.

## key_len 컬럼
쿼리를 처리하기 위해 다중 컬럼으로 구성된 인덱스에서 몇 개의 컬럼까지 사용했는지 알 수 있다.
인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값이다.
```sql
mysql> EXPLAIN
		SELECT * FROM dept_emp WHERE dept_no='d005';
        
```

두 개의 컬럼 (dpet_no, emp_no)으로 구성된 프라이머리 키를 가지는 dept_emp 테이블을 조회하는 쿼리이다.
이 쿼리는 dept_emp 테이블의 프라이머리 키 중에서 dept_no만 비교에 사용된다.
그래서 key_len 컬럼의 값이 16으로 표시된 것이다.
dept_no 컬럼의 타입이 CHAR(4)이기 때문에 프라이머리 키에서 앞쪽 16바이트만 유효하게 사용했다는 의미다.
실제 utf8mb4 문자 집합에서는 문자 하나가 차지하는 공간이 1바이트에서 4바이트까지 가변적이다. 하지만 MySQL 서버가 utf8mb4 문자를 위해 메모리 공간을 할당해야 할 때는 문자와 관계없이 고정적으로 4바이트로 계산한다.

> 테이블의 컬럼이 NULLABLE 컬럼으로 정의된 경우 MySQL에서는 NOT NULL이 아닌 컬럼에서는 컬럼의 값이 NULL인지 아닌지를 저장하기 위해 1바이트를 추가로 사용한다.

## ref 컬럼
접근 방법이 ref면 참조 조건(Equal 비교 조건)으로 어떤 값이 제공됐는지 보여준다.

사용자가 명시적으로 값을 변환하거나 내부적으로 값을 변활할 때 ref 칼럼에 func라고 출력된다.
예를 들어 문자 집합이 일치하지 않은 두 문자열 칼럼 조인, 숫자 타입의 컬럼과 문자 타입의 컬럼으로 조인할 때

가급적 조인 칼럼의 타입은 일치시키는 것이 좋다.

## rows 컬럼
MySQL 옵티마이저는 각 조건에 대해 가능한 처리 방식을 나열하고 각 처리 방식의 비용을 비교해 최종적으로는 하나의 실행 계획을 수립한다. 이때 각 처리 방식이 얼마나 많은 레코드를 읽고 비교해야 하는지 예측해서 비용을 산정한다. 대상 테이블에 얼마나 많은 레코드가 포함돼 있는지 또는 각 인덱스 값의 분포도가 어떤지를 통계 정보를 기준으로 조사해서 예측한다.

MySQL 실행 계획의 rows 컬럼 값은 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여준다. 이 값은 각 스토리지 엔진별로 가지고 있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해 낸 예상값이라서 정확하지는 않다. 또한 rows 컬럼에 표시되는 값은 반환하는 레코드의 예측치가 아니라 쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미한다. 그래서 실행 계획의 rows 컬럼에 출력되는 값과 실제 쿼리 결과 반환된 레코드 건수는 일치하지 않는 경우가 많다.

## filtered 컬럼
조인이 사용되는 경우에는 where 절에서 인덱스를 사용할 수 있는 조건도 중요 하지만 인덱스를 사용하지 못하는 조건에 일치하는 레코드 건수를 파악하는 것도 매우 중요하다.

filtered 칼럼의 값은 필터링되어 버려지는 레코드의 비율이 아니라 필터링되고 남은 레코드의 비율을 의미한다.

rows * (filtered / 100) = 수행한 레코드의 건수

일치하는 레코드 건수가 적은 테이블이 드라이빙 테이블이 되는 것이 조인의 횟수를 줄일 수 있어서 좋다.