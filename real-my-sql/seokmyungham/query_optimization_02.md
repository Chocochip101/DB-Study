# 쿼리 작성 및 최적화

## SELECT

웹 서비스와 같이 일반적인 OLTP 환경의 데이터베이스에서는 INSERT나 UPDATE 같은 작업은 거의 레코드 단위로 발생하므로
성능상 문제가 되는 경우는 별로 없다. 
  
하지만 SELECT는 여러 개의 테이블로부터 데이터를 조합해서 빠르게 가져와야 하기 때문에 여러 개의 테이블을 어떻게 읽을 것인가가
많이 중요하다. 그러려면 SELECT 쿼리 각 부분에 사용되는 기능이 성능적으로 어떻게 작용하는지 알아야한다.
  
![image](https://github.com/user-attachments/assets/a6755727-cd8f-43a4-962e-0ddadbf6883b)

위는 SQL 쿼리가 실행되는 순서다. 각 요소가 없는 경우는 가능하지만, 이 순서가 바뀌어서 실행되는 쿼리는 거의 없다.
또한 ORDER BY나 GROUP BY 절이 있더라도 인덱스를 이용해 처리할 때는 처리 단계가 생략된다.

![image](https://github.com/user-attachments/assets/324d8ae8-c265-4e9f-9c0e-9e2bb066b211)

ORDER BY가 사용된 쿼리에서 예외적인 순서로 실행되는 경우가 있다. JOIN이 동반된 쿼리일 경우 첫 번째 테이블만 읽어서
정렬을 수행한 뒤에 나머지 테이블을 읽을 수 있는데 주로 GROUP BY 절 없이 ORDER BY만 사용된 쿼리에서 사용될 수 있는 순서다.

#

### WHERE 절과 GROUP BY 절, ORDER BY 절의 인덱스 사용

WHERE 절이나 ORDER BY 또는 GROUP BY가 인덱스를 사용하려면 기본적으로 인덱스된 칼럼의 값 자체를 변환하지 않고
그대로 사용할 수 있어야 한다. 인덱스는 칼럼의 값을 아무런 변환 없이 B-TREE에 저장하기 때문에 WHERE, GROUP BY, ORDER BY 에서도
원본값을 검색하거나 정렬할 때만 B-TREE에 정렬된 인덱스를 이용할 수 있다.

예를 들면 아래와 같은 쿼리는 인덱스를 이용하지 못한다.

```sql
SELECT * FROM salaries WHERE salry*10 > 150000;
```

추가로 WHERE 절에 사용되는 비교 조건에서 연산자 양쪽의 두 비교 대상 데이터 타입이 일치해야 한다.

### WHERE 절의 인덱스 사용

WHERE 조건이 인덱스를 사용하는 방법은 `작업 범위 결정 조건`과 `체크 조건` 두 가지 방식으로 구분할 수 있다.

![image](https://github.com/user-attachments/assets/a37b013b-b132-4cf5-9907-49c03150c8c0)

WHERE 조건절의 순서는 실제 인덱스의 사용 여부와 무관하다. WHERE 조건절에 나열된 순서가 인덱스와 다르더라도
MySQL 서버 옵티마이저는 인덱스를 사용할 수 있는 조건들을 뽑아서 최적화를 수행할 수 있다.
  
COL_1과 COL_2는 동등 비교 조건이며, COL_3 은 범위 비교 조건이므로 COL_4의 조건은 단지 체크 조건으로 사용된다.
인덱스 순서상 COL_4의 직전 칼럼인 COL_3이 동등 비교 조건이 아니라 범위 비교 조건으로 사용됐기 때문이다.
  
8.0 버전부터는 다음 예제와 같이 인덱스를 구성하는 칼럼별로 정순과 역순 정렬을 혼합해서 생성할 수 있게 개선됐다.

```
ALTER TABLE ... ADD INDEX ix_col1234 (col_1 ASC, col_2 DESC, col_3 ASC, col_4 ASC);
```

AND 연산자와 달리 OR 연산자는 처리 방법이 완전히 달라진다.

```sql
SELECT *
FROM employees
WHERE first_name='Kebin' OR last_name='Poly';
```
위 쿼리에서 `first_name='Kebin'` 조건은 인덱스를 이용할 수 있지만 `last_name='Poly'`는 인덱스를 사용할 수 없다.
이 두 조건이 AND 연산자로 연결됐다면 first_name 인덱스를 이용하겠지만 OR 연산자로 연결됐기 때문에 결국 `풀 테이블 스캔`을
선택할 수 밖에 없다. (풀 테이블 스캔) + (인덱스 레인지 스캔)의 작업량 보다는 (풀 테이블 스캔) 한 번이 더 빠르기 때문이다.
  
이 경우 first_name과 last_name 칼럼에 각각 인덱스가 있다면 `index_merge` 접근 방법으로 실행할 수 있긴하다. 물론 풀 테이블 스캔보다는
빠르지만 여전히 제대로 된 인덱스 하나를 레인지 스캔하는 것 보다 느리다. WHERE 절에서 각 조건이 AND로 연결되면 읽어와야 할 레코드의 건수를 줄이는
역할을 하지만 각 조건이 OR로 연결되면 읽어서 비교해야 할 레코드가 더 늘어나기 때문에 OR 연산자가 있다면 주의해야 한다.

### GROUP BY 절의 인덱스 사용

GROUP BY 절에 명시된 칼럼의 순서가 인덱스를 구성하는 칼럼의 순서와 같으면 GROUP BY 절은 일단 인덱스를 이용할 수 있다.

- GROUP BY 절에 명시된 칼럼이 인덱스 칼럼의 순서와 위치가 같아야 한다.
- 인덱스를 구성하는 칼럼 중에서 뒤쪽에 있는 칼럼은 GROUP BY 절에 명시되지 않아도 인덱스를 사용할 수 있지만 인덱스의 앞족에 있는 칼럼이 GROUP BY 절에 명시되지 않으면 인덱스를 사용할 수 없다.
- WHERE 조건절과는 달리 GROUP BY 절에 명시된 칼럼이 하나라도 없으면 GROUP BY 절은 전혀 인덱스를 이용하지 못한다.

![image](https://github.com/user-attachments/assets/5e197ec4-793c-4554-a423-fe46bcd0aed7)

```SQL
GROUP BY COL_2, COL_1 
GROUP BY COL_1, COL_3, COL_2 
GROUP BY COL_1, COL_3 
GROUP BY COL_1, COL_2, COL_3, COL_4, COL_5
```

위 쿼리들은 모두 인덱스를 이용하지 못하는 경우들이다.

- 첫 번째와 두 번째 예제는 GROUP BY 칼럼이 인덱스를 구성하는 칼럼의 순서와 일치하지 않기 때문에 사용하지 못하 는 것이다.
- 세 번째 예제는 GROUP BY 절에 COL_3가 명시됐지만 COL_2가 그 앞에 명시되지 않았기 때문이다.
- 네 번째 예제에서는 GROUP BY 절의 마지막에 있는 COL_5가 인덱스에는 없어서 인덱스를 사용하지 못하는 것이다.

```SQL
GROUP BY COL_1 
GROUP BY COL_1, COL_2
GROUP BY COL_1, COL_2, COL_3 
GROUP BY COL_1, COL_2, COL_3, COL_4
```

```SQL
WHERE COL_1='상수' GROUP BY COL_2, COL_3 
WHERE COL_1='상수' AND COL_2='상수' GROUP BY COL_3, COL_4
WHERE COL_1='상수' AND COL_2='상수' AND COL_3='상수' GROUP BY COL_4
```

위 쿼리들은 인덱스를 사용할 수 있는 패턴이다.
WHERE 조건절에 COL_1이나 COL_2가 동등 비교 조건으로 사용된다면 GROUP BY 절에 COL_1이나 COL_2가 빠져도 인덱스를 이용한
GROUP BY가 가능할 때도 있다.

### ORDER BY 절의 인덱스 사용

MySQL에서 GROUP BY와 ORDER BY는 처리 방법이 상당히 흡사하다. 그래서 ORDER BY 절의 인덱스 적용 여부는 GROUP BY 요건과 거의 흡사하다.
그런데 ORDER BY는 추가로 정렬되는 각 칼럼의 오름차순 및 내림차순 옵션이 인덱스와 같거나 정반대인 경우에만 사용할 수 있다.

![image](https://github.com/user-attachments/assets/4c3d8072-a8ee-4c41-a145-9efc30e9f06b)

인덱스의 모든 칼럼이 ORDER BY 절에 사용돼야 하는 것은 아니지만, ORDER BY 절의 칼럼들이 인덱스에 정의된 칼럼의 왼쪽부터 일치해야 한다.

```SQL
ORDER BY COL_2, COL_3 
ORDER BY COL_1, COL_3, COL_2 
ORDER BY COL_1, COL_2 DESC, COL_3 
ORDER BY COL_1, COL_3
ORDER BY COL_1, COL_2, COL_3, COL_4, COL_5
```

위 쿼리들은 모두 인덱스를 이용할 수 없다. 정렬 순서가 생략되면 오름차순으로 해석한다.

- 첫 번째 예제는 인덱스의 제일 앞쪽 칼럼인 COL_1이 ORDER BY 절에 명시되지 않았기 때문에 인덱스를 사용할 수 없다.
- 두 번째 예제는 인덱스와 ORDER BY 절의 칼럼 순서가 일치하지 않기 때문에 인덱스를 사용할 수 없다.
- 세 번째 예제는 ORDER BY 절의 다른 칼럼은 모두 오름차순인데, 두 번째 칼럼인 COL_2의 정렬 순서가 내림차순이라 서 인덱스를 사용할 수 없다. 인덱스가 "(COL_1 ASC, COL_2 DESC, COL_3 ASC, COL_4 ASC)"와 같이 정의됐다 면 이 정렬은 인덱스를 사용할 수 있게 된다.
- 네 번째 예제는 인덱스에는 COL_ 1과 COL_3 사이에 COL_2 칼럼이 있지만 ORDER BY 절에는 COL.2 칼럼이 명시되지 않았기 때문에 인덱스를 사용할 수 없다.
- 다섯 번째 예제는 인덱스에 존재하지 않는 COL_5가 ORDER BY 절에 명시됐기 때문에 인덱스를 사용하지 못한다.

### WHERE 조건과 ORDER BY(또는 GROUP BY) 절의 인덱스 사용

일반적으로 우리가 사용하는 쿼리는 WHERE 절을 가지고 있으며, 선택적으로 ORDER BY, GROUP BY 절을 포함할 것이다.
쿼리에 WHERE 절만 또는 GROUP BY, ORDER BY 절만 포함돼 있다면 사용된 절 하나에만 초점을 맞춰서 인덱스를 사용할 수 있게 튜닝하면 된다.
  
하지만 애플리케이션에서 사용되는 쿼리는 그렇게 단순하지 않다. SQL 문장이 WHERE 절과 ORDER BY절을 가지고 있다고 가정했을 때 WHERE 조건은
A 인덱스를 사용하고 ORDER BY는 B 인덱스를 사용하도록 쿼리가 실행될 수는 없다.

- WHERE 절과 ORDER BY 절이 동시에 같은 인덱스를 이용
    - WHERE 절의 비교 조건에서 사용하는 칼럼과 ORDER BY 절의 `정렬 대상 칼럼이 모두 하나의 인덱스에 연속해서 포함`돼 있을 때 이 방식으로 인덱스를 사용할 수 있다. 이 방법 은 나머지 2가지 방식보다 훨씬 빠른 성능을 보이기 때문에 가능하다면 이 방식으로 처리할 수 있게 쿼리를 튜닝하 거나 인덱스를 생성하는 것이 좋다.
- WHERE 절만 인덱스를 이용
    - ORDER BY 절은 인덱스를 이용한 정렬이 불가능하며, 인덱스를 통해 검색된 결과 레코드를 별도의 정렬 처리 과정(Using Filesort)을 거쳐 정렬을 수행한다. 주로 이 방법은 WHERE 절의 조건에 일치하는 `레코드의 건수가 많지 않을 때` 효율적인 방식이다.
- ORDER BY 절만 인덱스를 이용
    - ORDER BY 절은 인덱스를 이용해 처리하지만 WHERE 절은 인덱스를 이용하지 못한다. 이 방식은 ORDER BY 절의 순서대로 인덱스를 읽으면서 레코드 한 건씩 WHERE 절의 조건에 일치하는지 비교하고, 일치하지 않을 때는 버리는 형태로 처리한다. 주로 `아주 많은 레코드를 조회해서 정렬해야 할 때`는 이런 형태로 튜닝 하기도 한다.
 
또한 WHERE 절에서 동등 비교 조건으로 비교된 칼럼과 ORDER BY 절에 명시된 칼럼이 순서대로 빠짐없이 인덱스 칼럼의 왼쪽부터 일치해야 한다.
8.0 버전에 새롭게 추가된 인덱스 스킵 스캔 최적화는 인덱스에 나열된 칼럼 순서상 선행되는 칼럼의 조건이 없다고 하더라도 인덱스의 후행 칼럼을 이용할 수 있게 해주긴 한다.

```SQL
SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_1, COL_2, COL_3;
SELECT * FROM tb_test WHERE COL_1 > 10 ORDER BY COL_2, COL_3;
```

첫 번째 예제 쿼리에서 COL_1 > 10 조건을 만족하는 COL_1 값은 여러 개일 수 있다. 
하지만 ORDER BY 절에 COL_1부터 COL_3까지 순서대로 명시됐기 때문에 인덱스를 사용해 WHERE 조건절과 ORDER BY 절을 처리할 수 있다.
  
하지만 두 번째 쿼리에서는 WHERE 절에서 COL_1 동등 조건이 아니라 범위 조건으로 검색됐는데 ORDER BY 절에 COL_1이 명시되지 않았기 때문에
정렬할 때는 인덱스를 이용할 수 없게된다.

### GROUP BY 절과 ORDER BY 절의 인덱스 사용

GROUP BY와 ORDER BY 절이 동시에 사용된 쿼리에서 두 절이 모두 하나의 인덱스를 사용해서 처리되려면 
GROUP BY 절에 명시된 칼럼과 ORDER BY에 명시된 칼럼의 순서와 내용이 모두 같아야 한다.
  
GROUP BY와 ORDER BY가 같이 사용된 쿼리에서는 둘 중 하나라도 인덱스를 이용할 수 없을 때는 둘 다 인덱스를 사용하지 못한다.
GROUP BY는 인덱스를 이용할 수 있지만 ORDER BY가 인덱스를 이용할 수 없을 때 GROUP BY, ORDER BY절이 모두 인덱스를 이용하지 못한다는 것이다.

```SQL
GROUP BY COL_1, COL_2 ORDER BY COL_2
GROUP BY COL_1, COL_2 ORDER BY COL_1, COL_3
```

### Short-Circuit Evaluation

여러 개의 표현식이 AND 또는 OR 논리 연산자로 연결된 경우 선행 표현식의 결과에 따라 후행 표현식을 평가할지 말지 결정하는 최적화를
`Short-circuit Evalutaion`이라고 한다.
  
MySQL 서버는 쿼리의 WHERE 절에 나열된 조건을 순서대로 `Short-circuit Evalutaion` 방식으로 평가해서 해당 레코드를 반환해야 할지 말지를 결정한다. 즉 MySQL 서버에서 쿼리를 작성할 때 가능하면 복잡한 연산 또는 다른 테이블의 레코드를 읽어야 하는 서브쿼리 조건 등은 WHERE 절의 뒤쪽으로 배치하는 것이 성능상 도움이 된다. 
  
물론 WHERE 조건에서 인덱스를 사용할 수 있는 조건은 WHERE 절의 어느 위치에 나열되든지 그 순서에 관계없이 가장 먼저 평갸되기 때문에 고려하지 않아도 된다.

### LIMIT

쿼리 문장에 GROUP BY나 ORDER BY 같은 전체 범위 작업이 선행되더라도 LIMIT 절이 있다면 크진 않지만 나름의 성능 향상을 볼 수 있다.
ORDER BY, GROUP BY, DISTINCT가 인덱스를 이용해 처리될 수 있다면 LIMIT 절은 꼭 필요한 만큼의 레코드만 읽게 만들어주기 때문에 쿼리의 작업량을 상당히 줄여준다. 또는 정렬, 그루핑, 중복 제거가 없는 쿼리에서 LIMIT 조건은 쿼리가 상당히 빨리 끝나게 만들어줄 수도 있다.
  
그런데 LIMIT를 사용할 때 주의해야 할 점은 사용자 화면에 레코드가 몇 건이 출력되느냐보다 MySQL 서버가 그 결과를 만들어 내기 위해 어떠한 작업들을 했는지가 중요하다. 특히 많은 응용 프로그램에서 페이징 처리를 할 때 LIMIT를 사용하는 경우가 많다. 일반적인 쿼리에 경우에 특별히 문제가 되지는 않는다. 하지만 `LIMIT n, m`의 n과 m이 매우 커질 수가 있는데 쿼리 실행에 상당히 오랜 시간이 걸린다.

```sql
mysql> SELECT * FROM salaries ORDER BY salary LIMIT 0, 10;
mysql> SELECT * FROM salaries ORDER BY salary LIMIT 2000000, 10;
```

`LIMIT 2000000, 10`은 먼저 salaries 테이블을 처음부터 읽으면서 2000010건의 레코드를 읽은 후 2000000건은 버리고 마지막 10건만 사용자에게 반환한다. 실제 사용자의 화면에는 10건만 표시되지만, MySQL. 서버는 2000010건의 레코드를 읽어야 하기 때문에 쿼리가 느려지는 것이다.

LIMIT 조건의 페이징이 처음 몇 개 페이지 조회로 끝나지 않을 가능성이 높다면 WHERE 조건절로 읽어야 할 위치를 찾고 그 위치에서 10개만 읽는 형태의 쿼리를 사용하는 것이 좋다.

### COUNT

COUNT() 함수는 결과 레코드의 건수를 반환하는 함수다. 여기서 *는 모든 칼럼을 가져오는 의미가 아니라 그냥 레코드 자체를 의미한다. 그래서 `COUNT(*)`라고 해서 레코드의 모든 칼럼을 읽는 형태로 처리하지는 않는다. 그래서 `COUNT(PK)`, `COUNT(1)`과 같이 사용하지 않고 `COUNT(*)`라고 표현해도 동일한 처리 성능을 보인다.
  
InnoDB 스토리지 엔진을 사용하는 테이블에서는 WHERE 조건이 없는 COUNT(*) 쿼리라고 하더라도 직접 데이터나 인덱스를 읽어야만 레코드 건수를 가져올 수 있기 때문에 큰 테이블에서 COUNT() 함수를 사용하는 작업은 주의해야 한다.
  
`COUNT(*)` 쿼리에서 가장 많이 하는 실수는 `ORDER BY` 구문이나 `LEFT JOIN` 처럼 레코드 건수를 가져오는 것과는 무관한 작업을 포함하는 것이다. 대부분 `COUNT(*)` 쿼리는 페이징 처리를 위해 사용할 때가 많은데 많은 개발자가 SELECT 쿼리를 그대로 복사해서 칼럼이 명시된 부분만 삭제하고 그 부분을 `COUNT(*)` 함수로 대체해서 사용할 떄가 많다. `COUNT(*)` 쿼리에서 ORDER BY 절은 어떤 경우에도 필요치 않다. 그리고 LEFT JOIN 또한 레코드 건수의 변화가 없거나 아우터 테이블에서 별도의 체크를 하지 않아도 되는 경우에는 모두 제거하는 것이 성능상 좋다.

> 8.0 버전부터 SELECT COUNT(*) 쿼리에 사용된 ORDER BY 절은 옵티마이저가 무시하도록 개선됐다.
> 그러나 여전히 꼭 필요한 부분만 간결하게 사용해 쿼리를 작성하는 것은 쿼리의 복잡도를 낮추고 가독성을 높인다는 장점이 있다.

많은 사용자가 일반적으로 칼럼의 값을 SELECT 하는 쿼리보다 `COUNT(*)` 쿼리가 훨씬 빠르게 실행될 것으로 생각한다. 하지만 인덱스를
제대로 사용하도록 튜닝되지 못한 COUNT 쿼리는 페이징해서 데이터를 가져오는 쿼리보다 몇십 배 느리게 실행될 수도 있다.
`COUNT(*)` 쿼리도 많은 부하를 일으키기 때문에 주의 깊게 작성해야 한다.

게시물 목록을 보여줄 때 일반적으로 다음과 같은 쿼리를 사용한다.
```SQL
게시물 건수 확인
SELECT COUNT(*) FROM articles WHERE borad_id=1

특정 페이지의 게시물 조회
SELECT * FROM articles WHERE borad_id=? ORDER BY article_id DESC LIMIT 0, 10
```

첫 번째 쿼리는 board_id가 1인 레코드 건수가 백만 건이라면 첫 번째 쿼리는 articles 테이블에서 100만건을 일겅야 한다.
물론 커버링 인덱스가 가능하다면 쿼리 성능은 빨라질 것이다. 하지만 실제 서비스에서는 보여줄지 말지, 그리고 삭제됐는지 여부 등을 식별해야 하기 때문에 테이블의 데이터를 읽어야 하는 경우가 대부분이다. 그렇다면 건수를 확인하는 첫 번째 쿼리의 부하는 게시물 10건을 가져오는 두 번째 쿼리보다 10만 배 느리게 처리될 것이다.
  
물론 게시물을 가져오는 두 번째 쿼리도 다음 페이지로 넘어가면 갈수록 성능이 더 느려질 가승넝이 높다. 게시물 전체 건수를 조회하는 작업은 피하는 것이 좋다. 게시물의 페이지 번호를 보여주는 방식보다는 "이전", "다음" 버튼만 표시하는 방식을 검토해볼 것을 권장한다.


## Reference 

**위 글은 책 RealMySQL 8.0 2권을 구입하여 읽고 정리한 내용입니다.**
- [도서 홈페이지 https://wikibook.co.kr/realmysql802/](https://wikibook.co.kr/realmysql802/)
