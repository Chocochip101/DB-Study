# 정규화

## 이상(anomaly) 현상

불필요한 데이터 중복으로 인해 릴레이션에 대한 데이터 삽입,수정,삭제 연산을 수행할 때 발생할 수 있는 부작용을 말한다.  

## 이상 현상의 종류

### 삽입 이상(insertion anomaly)

릴레이션에 새 데이터를 삽입하려면 `불필요한 데이터도 함께 삽입`해야 하는 문제이다.

![image](https://github.com/seokmyungham/DB-Study/assets/97608735/10668743-5224-49be-845b-a23a99663282)

위 이벤트참여 릴레이션은 현재 삽입 이상이 발생하는 상태이다.  
아직 이벤트에 참여하지 않은 (아이디: "loki", 이름: "로키", 등급: "gold") 신규 고객의 데이터를 릴레이션에 삽입할 수 없고 
삽입하려면 실제로 참여하지 않은 `임시 이벤트 번호`를 삽입해야 한다.

### 갱신 이상(update anomaly)

릴레이션의 중복된 튜플들 중 `일부만 수정하여 데이터가 불일치`하게 되는 모순이 발생하는 문제이다.

![image](https://github.com/seokmyungham/DB-Study/assets/97608735/2360f18f-edec-4402-a634-9f1ae75fde2e)

### 삭제 이상(deletion anomaly)

릴레이션에서 투플을 삭제하면 `꼭 필요한 데이터까지 손실`되는 연쇄 삭제 현상이 발생하는 문제이다.

위 릴레이션에서 아이디가 "orange"인 고객이 이벤트 참여를 취소해 관련 튜플을 삭제하게되면 
이벤트 참여와 관련이 없는 고객아이디, 고객이름, 등급 데이터까지 손실된다.

#

## 정규화의 필요성

이상 현상이 발생하지 않도록, 릴레이션을 관련 있는 속성들롤만 구성하기 위해 릴레이션을 분해하는 과정이다.  
`함수적 종속성`(속성들 간의 관련성)을 판단하여 정규화를 수행한다.

기본 목표: 관련이 없는 함수 종속성은 별개의 릴레이션으로 표현

### 함수 종속(Functional Dependency)

`X가 Y를 함수적으로 결정한다.` 
  
속성 자체의 특성과 의미를 기반으로 함수 종속성을 판단해야 함.  
  - 속성 값은 계속 변할 수 있으므로 현재 릴레이션에 포함된 속성 값만으로 판단하면 안 된다.
  
일반적으로 기본키와 후보키는 릴레이션의 다른 모든 속성들을 함수적으로 결정한다.  
기본키나 후보키가 아니어도 다른 속성 값을 유일하게 결정하는 속성은 함수 종속 관계에서 결정자가 될 수 있다.

![image](https://github.com/seokmyungham/DB-Study/assets/97608735/6f121ae9-05bd-4502-bc06-d894d1b4529d)

- `완전 함수 종속`(FFD: Full Functional Dependency)
  - 릴레이션에서 속성 집합 Y가 속성 집합 X에 함수적으로 종속되어 있으며, 일부분에 종속된 것이 아닌 전체에 종속됨을 의미
  - 일반적으로 함수 종속은 완전 함수 종속을 의미한다.
  - `당첨여부` -> {`고객아이디`, `이벤트번호`}에 완전 함수 종속
- `부분 함수 종속`(PFD: Partial Functional Dependency)
  - 릴레이션에서 속성 집합 Y가 속성 집합 X의 전체가 아닌 일부분에도 함수적으로 종속됨을 의미
  - 일반적이지 않은 경우이다.
  - `고객이름` -> {`고객아이디`, `이벤트번호`}에 부분 함수 종속
- `이행적 함수 종속`(transitive FD)
  - 릴레이션을 구성하는 3개의 속성 집합 X, Y, Z에 대해 함수 종속 관계가 X -> Y, Y -> Z가 존재하면 논리적으로 X -> Z가 성립되는데 이 때 Z가 X에 이행적으로 함수 종속되었다고 한다.

### 주의 사항

정규화를 통해 릴레이션은 `무손실 분해`되어야 한다.
- 릴레이션이 의미상 동등한 릴레이션들로 분해되어야 하고, 분해로 인한 정보 손실이 발생하지 않아야 힘
- 분해된 릴레이션들을 자연 조인하면 분해 전의 릴레이션으로 복원 가능해야 한다.

---

## 정규형

![image](https://github.com/seokmyungham/DB-Study/assets/97608735/f5f9815c-aab5-4ed7-9788-570e07b9e21b)

- 정규형은 릴레이션이 정규화된 정도를 의미  
- 각 정규형마다 제약조건이 존재하는데, 정규형의 차수가 높아질수록 요구되는 제약조건이 많아지고 엄격해진다.
- 릴레이션의 특성을 고려해서 적합한 정규형을 선택하는 것이 중요


### 제 1정규형 (1NF; First Normal Form)
![image](https://github.com/seokmyungham/DB-Study/assets/97608735/6305ecab-2b8b-4ac1-b861-212afcef721b)
![image](https://github.com/seokmyungham/DB-Study/assets/97608735/c347dba9-e2f0-431a-829c-919fe034f1ef)

릴레이션의 모든 속성이 더는 분해되지 않는 `원자 값(atomic value)`만 가지면 제 1정규형을 만족한다.  
제 1정규형을 만족함에도 `부분 함수 종속`으로 인한 이상 현상이 발생한다.
- 삽입 이상, 갱신 이상, 삭제 이상 모두 발생
    - 발생이유: 기본 키인 {고객아이디, 이벤트번호}의 일부분인 고객아이디에만 종속되는 등급과 할인율 속성이 존재하기 때문
 
#
 
### 제 2정규형 (2NF; Second Normal Form)

릴레이션이 제 1정규형에 속하고, 기본키가 아닌 모든 속성이 기본키에 `완전 함수 종속`되면 제 2정규형을 만족한다.  
- 정규화 과정: 부분 함수 종속을 제거하고 모든 속성이 기본키에 완전 함수 종속되도록 릴레이션을 분해

제 2정규형을 만족함에도 여전히 `이행적 함수 종속`으로 인한 이상 현상이 발생한다.
- 삽입 이상, 갱신 이상, 삭제 이상 모두 발생
  - 발생이유: 고객아이디가 등급을 통해 할인율을 결정하는 이행적 함수 종속 관계가 존재하기 때문

#

### 제 3정규형 (3NF; Third Normal Form)

릴레이션이 제 2정규형에 속하고 기본키가 아닌 모든 속성이 기본키에 이행적 함수 종속이 되지 않으면 제 3정규형을 만족한다.  
- 정규화 과정: 모든 속성이 기본키에 이행적 함수 종속이 되지 않도록 릴레이션을 분해

#

### 보이스/코드 정규형 (BCNF; Boyce-Codd Normal Form)

하나의 릴레이션에 여러 개의 후보키가 존재하는 경우, 제 3정규형까지 모두 만족해도 이상 현상이 발생할 수 있다.  
특정 릴레이션의 함수 종속 관계에서 `모든 결정자가 후보키가 아니면` 보이스/코드 정규형을 만족할 수 없다.

![image](https://github.com/seokmyungham/DB-Study/assets/97608735/3cef03ee-a620-40ed-b8bd-1838ced7f549)

위 릴레이션에서 이상 현상이 발생하는 이유는 담당강사번호가 후보키가 아님에도 인터넷강좌 속성을 결정하기 때문이다.  
현재 릴레이션의 후보키는 {`고객아이디`, `인터넷강좌`} , {`고객아이디`, `담당강사번호`} 임에도 불구하고  
`담당강사번호`가 `인터넷강좌` 속성에 대해 결정자 역할을 하고 있다.  

- 삽입 이상: `담당강사번호`가 `인터넷강좌` 속성을 결정하지만 새로운 정보를 추가하려면 고객아이디가 필요하다.
- 갱신 이상: `담당강사번호`가 맡는 `인터넷강좌` 속성을 하나만 변경하면 데이터 불일치가 발생한다.
- 삭제 이상: `고객아이디`가 수강하는 `인터넷강좌` 속성을 제거하면 `담당강사번호`가 `인터넷강좌`를 담당하는 정보도 같이 사라진다.

- 정규화 과정: 후보키가 아닌 결정자를 제거하도록 릴레이션을 분해

#

### 제 4정규형

릴레이션이 보이스/코드 정규형을 만족하면서, 함수 종속이 아닌 다치 종속(MVD; MultiValued Dependency)을 제거하면 제 4정규형에 속한다.

### 제 5정규형

릴레이션이 제 4정규형을 만족하면서, 후보키를 통하지 않는 조인 종속(JD; Join Dependency)을 제거하면 제 5정규형에 속한다.

### 정규화 시 주의사항

- 모든 릴레이션이 제 5정규형에 속해야만 바람직한 것은 아니다.
- 일반적으로 제 3정규형이나 보이스/코드 정규형에 속하도록 릴레이션을 분해하여 데이터 중복을 줄이고 이상 현상을 해결하는 경우가 많다.
