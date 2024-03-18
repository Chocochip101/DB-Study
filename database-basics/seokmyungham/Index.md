# 인덱스

데이터베이스 인덱스는 원하는 데이터에 빠르게 접근할 수 있도록 검색 속도를 향상시키는 데 사용되는 자료구조이다. 

데이터베이스에서 테이블의 전체 데이터를 순차적으로 검색하는 대신 인덱스 자료구조를 사용하면 조회 성능을 드라마틱하게 향상시킬 수 있다. 마치 책의 색인 역할과 유사하다.

#

하지만 인덱스를 관리하기 위한 추가적인 오버헤드도 존재한다.

인덱스가 적용된 컬럼에 삽입, 수정, 삭제 연산이 수행된다면 각각 인덱스를 유지하기 위한 `추가적인 연산`이 실행되게 된다.

특히 컬럼을 수정, 삭제 한다고 인덱스를 같이 삭제하는 것이 아니기 때문에 인덱스가 적용된 테이블에 수정, 삭제 연산이 빈번하게 발생한다면 실제 데이터보다 인덱스가 훨씬 더 많이 존재하게 되어 오히려 성능이 떨어지게 될 것이다.

따라서 자주 조회되는 열이나 Join 조건에 사용되는 열에 인덱스를 생성하는 것이 중요하다.

## 인덱스 자료 구조

### Hash Table

해시 테이블은 Key 값을 해시 함수를 통해 해시 값으로 변환하고, 해시값을 기반으로 데이터를 저장하는 자료구조이다.

해시 테이블은 빠른 데이터 검색이 필요할 때 유용하며 Key 값을 이용해서 고유한 index를 생성해서 index에 저장된 값을 꺼내오도록 한다.

하지만 해시 테이블은 부등호 연산(>, <)이 자주 사용되는 데이터 베이스 대신 등호(=)연산이 자주 사용되는 조회에 특화되어있다.

해시 테이블은 해시 값에 기반하여 데이터를 저장하기 때문에 데이터가 순서대로 정렬되어 있지 않기 때문이다. 따라서 범위 검색이 필요한 경우는 해시 테이블이 적절하지 않을 수 있다.


### B+ Tree

B+ Tree는 데이터를 계층적으로 정렬하여 저장하는 균형 트리 자료구조이다. 빠른 검색, 삽입, 삭제 작업을 지원한다.

B+ Tree는 데이터베이스의 인덱스를 위해 자식 노드가 2개 이상인 B-Tree를 개선시킨 구조인데 B+Tree는 리프노드에만 인덱스와 함께 데이터를 가지고 있는 형태를 띈다.

루트 노드와 경로에 존재하는 노드에는 키 값이 저장되어 있고, 리프 노드들은 연결 리스트로 연결되어 있기 때문에 루트 노드에서 특정 리프 노드에 이르는 한 개의 경로만 검색하면 된다. 
> O(log n)의 시간 복잡도

따라서 범위 검색이 빈번하게 일어나는 일반적인 데이터베이스의 경우 Hash Table보다는 B+ Tree 자료 구조가 인덱스에 매우 특화되어 있다.