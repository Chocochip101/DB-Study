
## 정규화

### anomaly (이상현상)
- 데이터 조작 작업에 따라 테이블의 일관성을 훼손하여 데이터의 무결성을 깨뜨리는 현상
- insert anomaly : 투플 삽입시 Null 이 입력되거나 테이블에 중복데이터가 삽입되어 데이터의 중복성이 증가
- delete anomaly : 투플 삭제 시 필요한 데이터가 함께 삭제되는 연쇄 삭제 현상이 발생할 수 있다.
- update anomaly : 투플 수정시 테이블의 속선 간 일관성이 깨질 수 있다.

### functional dependency (FD)
- A -> B : "A가 B를 결정한다", "B는 A에 종속된다.", "A는 B의 결정자다."

- X -> X 이다.
- (trivial, subset 부분집합 규칙) :If Y ⊆ X, then X -> Y
- (augmentation, 증가 규칙) : If X -> Y then XZ -> YZ
- (transitivity, 이행 규칙) : If X -> Y and Y -> Z then, X -> Z
- (union, 결합 규칙) : If X -> Y and X -> Z then, X -> YZ
- (decomposition, 분해 규칙) : If X -> YZ, then X -> Y and X -> Z
- (pseudotransitivity, 유사이행 규칙) : If X -> Y and WY -> Z, then WX -> Z

- K가 R(K, A, B, C) 의 기본키라면 K -> A, K -> B, K -> C 가 성립한다.
- K -> KABC도 성립하므로 K -> R 이다.

- 이상현상은 한 개의 릴레이션에 두 개 이상의 정보가 포함되어 있을 때 나타난다.
- 즉 결정자가 두개 이상 있으면 이상현상이 발생한다.

### 정규화
- 이상현상이 발생하는 릴레이션을 분해하여 이상현상을 없애는 과정
- 중복된 데이터를 허용하지 않음으로써 무결성(Integrity)를 유지할 수 있으며, 데이터베이스의 저장 용량 역시 줄일 수 있음

- First Normal Form : 모든 컬럼이 원자값(Atomic Value, 하나의 값)을 갖는 경우
- Second Normal Form : 제1 정규형을 만족하고, 기본키가 아닌 속성이 기본키에 완전 함수 종속일 때 (partial functional dependency가 없는 경우)
- Third Normal Form : 제2 정규형을 만족하고 이행적 종속이 없는 경우
- BCNF Normal Form : 함수 종속성 X → Y가 성립할 때 모든 결정자 X가 후보키인 정규형

- 무손실 분해 (Loseless-join decomposition) : 다시 조인 연산을 했을 때 데이터 손실이 없는 것
