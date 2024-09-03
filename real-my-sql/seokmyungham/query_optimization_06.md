# 쿼리 작성 및 최적화

## Update

### UPDATE ... ORDER BY ... LIMIT n

UPDATE와 DELETE는 조건절에 일치하는 모든 레코드를 업데이트하는 것이 일반적이지만, MySQL에서는 UPDATE DELETE 문장에 ORDER BY 절과 LIMIT 절을 동시에 사용해 특정 칼럼으로 정렬해서 상위 몇 건만 변경 및 삭제하는 것도 가능하다. 한 번에 너무 많은 레코드를 변경 및 삭제하는 작업은 서버에 과부하를 유발하거나 다른 커넥션의 쿼리 처리를 방해할 수도 있다. 이 때 LIMIT을 이용해 조금씩 변경하거나 삭제하는 방식을 손쉽게 구현할 수 있다.
  
하지만 복세 소스 서버에서 ORDER BY ... LIMIT이 포함된 UPDATE나 DELETE 문장을 실행하면 경고 메시지가 발생할 수 있다. 바이너리 로그 포맷이 STATEMENT 기반의 복제에서는 주의가 필요하다. ORDER BY에 의해 정렬되더라도 중복된 값의 순서가 복제 소스 서버와 레플리카 서버에서 달라질 수 있기 때문인데, PK로 정렬하면 문제는 없지만 여전히 경고 메시지는 기록된다. 복제가 구축된 MySQL 서버에서 ORDER BY가 포함된 UPDATE나 DELETE 문장을 사용할 때는 주의해야한다.

### JOIN UPDATE

두 개 이상의 테이블을 조인해 조인된 결과 레코드를 변경 및 삭제하는 쿼리를 JOIN UPDATE라고 한다. 조인된 테이블에서 특정 테이블의 칼럼 값을 다른 테이블의 칼럼에 업데이트해야 할 때 주로 조인 업데이트를 사용한다.
  
일반적으로 JOIN UPDATE는 조인되는 모든 테이블에 대해 읽기 참조만 되는 테이블에 읽기 잠금이 걸리고, 칼럼이 변경되는 테이블은 쓰기 잠금이 걸린다. OLTP 환경에서 JOIN UPDATE가 데드락을 유발할 가능성이 높으므로 너무 빈번하게 사용하는 것은 피하는 것이 좋다. 배치 프로그램이나 통계용 UPDATE 문장에서는 유용하게 사용할 수 있다.

```SQL
UPDATE departments d, dept_emp de
SET d.emp_count=COUNT(*)
WHERE de.dept_no=d.dept_no
GROUP BY de.dept_no;
```

위의 JOIN UPDATE는 dept_emp 테이블에서 부서별로 사원의 수를 departments 테이블의 emp_count 칼럼에 업데이트하기 위해 만든 쿼리다. dept_emp 테이블에서 부서별로 사원의 수를 가져오기 위해 GROUP BY가 사용된 것을 알 수 있다. 하지만 JOIN UPDATE 문장은 GROUP BY나 ORDER BY를 지원하지 않기때문에 동작하지 않고 에러를 발생시키게 된다.

```SQL
UPDATE departments d,
          (SELECT de.dept_no, COUNT(*) AS emp_count
           FROM dept_emp de
           GROUP BY de.dept_no) dc
SET d.emp_count=dc.emp_count
WHERE dc.dept_no=d.dept_no;

UPDATE (SELECT de.dept_no, COUNT(*) AS emp_count
        FROM dept_emp de
        GROUP BY de.dept_no) dc
STRAIGHT_JOIN departments d ON dc.dept_no=d.dept_no
SET d.emp_count=dc.emp_count;

UPDATE /*+ JOIN ORDER (dc, d) */
(SELECT de.dept_no, COUNT(*) AS emp_count
FROM dept_emp de
GROUP BY de.dept_no ) dc
INNER JOIN departments d ON dc.dept_no=d.dept_no
SET d.emp_count=dc.emp_count;
```

JOIN UPDATE를 서브쿼리로 개선한 것이다. 일반 테이블이 조인될 때는 임시 테이블이 드라이빙 테이블이 되는 것이 일반적으로 빠른 성능을 보여준다. 옵티마이저가 최적의 조인 방향을 알아서 선택하지만 조인의 방향을 명시적으로 설정하고 싶으면 JOIN UPDATE 문장에 STRAIGHT_JOIN 키워드를 사용하면 된다. 또는 JOIN_ORDER 옵티마이저 힌트를 사용하자
  
STRAIGHT_JOIN 키워드는 조인의 순서를 지정하는 힌트이기도 하지만 INNER JOIN 또는 LEFT JOIN과 같이 조인 키워드로 사용하기도 한다. INNER JOIN 이나 LEFT JOIN 키워드는 사실 테이블의 조인 순서를 결정하는 키워드는 아니다. 하지만 STRAIGHT_JOIN 키워드는 조인의 순서까지 결정한다. 키워드 왼쪽에 명시된 테이블이 드라이빙 테이블이 되며 오른쪽이 드리븐 테이블이 된다.

## DELETE

### JOIN DELETE 

## Reference 

**위 글은 책 RealMySQL 8.0 2권을 구입하여 읽고 정리한 내용입니다.**
