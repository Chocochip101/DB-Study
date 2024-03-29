## 키
RDBMS에서 키는 릴레이션에서 특정 튜플을 식별할 때 사용하는 속성 혹은 속성의 집합이다. 릴레이션은 중복된 튜플을 허용하지 않기 때문에 각각의 튜플에 포함된 속성들 중 하나는 값이 달라야한다. 즉 키가 되는 속성(혹은 속성의 집합)은 반드시 값이 달라서 튜플들을 서로 구별할 수 있어야 한다.

키는 각 릴레이션의 튜플을 유일하게 식별하는 장치이며 동시에 각 릴레이션 간의 관계를 말해주는 연결고리다.

### 슈퍼키
**슈퍼키는 튜플을 유일하게 식별할 수 있는 하나의 속성 혹은 속성의 집합을 말한다.**

튜플을 유일하게 식별할 수 있는 값이면 모두 슈퍼키가 될 수 있다. 슈퍼키는 포함하지 않아도 되는 속성을 포함할 수 있다.
- (주민번호), (주민번호, 이름), (주민번호, 이름, 주소), ...

우리가 관심을 두어야할 것은 튜플을 식별할 수 있는 최소한의 속성 집합이다. 키를 구성하는 **속성이 많으면 그만큼 관계 표현이 복잡해지고 사용에도 불편이 따른다.**

### 후보키
후보키는 튜플을 유일하게 식별할 수 있는 속성의 **최소** 집합이다. 예를 들어 (주민번호, 이름)은 슈퍼키이지만 후보키는 아니다. 

### 기본키
기본키는 여러 후보키 중 하나를 선정하여 대표로 삼는 키를 말한다. 후보키가 하나뿐이라면 그 후보키는 사용하면 되고 여러 개라면 릴레이션의 특정을 반영하여 하나를 선택하면 된다.

#### 기본키 선정 고려사항
- 릴레이션 내 튜플을 식별할 수 있는 고유한 값을 가져야한다.
- NULL 값은 허용하지 않는다.
- 키 값의 변동이 일어나지 않아야 한다.
- 최대한 적은 수의 속성을 가져야 한다.
- 키를 사용하는데 있어 문제 발생 소지가 적어야 한다.


### 대리키
기본키가 보안을 요하거나, 마땅한 기본키가 없을 경우 **가상의 속성을 만들어 기본키로 삼는 경우**가 있다. 이러한 키를 **대리키** 또는 **인조키**라고 한다. 대리키는 DBMS나 관련 SW에서 임의로 생성하는 값으로 사용자가 직접 그 값의 의미를 알 수 없다.

### 대체키
대체키는 기본키로 선정되지 않은 후보키를 말한다.

### 외래키
외래키는 다른 릴레이션의 기본키를 참조하는 속성을 말한다. 외래키는 다른 릴레이션의 기본키를 참조하여 데이터 모델의 특징인 릴레이션 간의 관계를 표현한다.

외래키가 성립하기 위해서는 참조하고 참조되는 양쪽 릴레이션의 도메인이 서로 같아야 한다. 또한 **참조되는 릴레이션의 기본키 값이 변경되면 이 기본키를 참조하는 외래키 값 역시 변경**되어야 한다. 참조하는 외래키 값이 참조되는 기본키 값에 연동된다는 의미로, 외래키는 항상 데이터의 일관성을 유지해야 한다. 이러한 특징을 외래키 제약조건이라고 한다. 외래키(참조하는 키)는 참조되는 릴레이션의 기본키와 달리 NULL 값을 포함할 수 있고 중복 값도 허용한다.

외래키 사용 시 참조하는 릴레이션과 참조되는 릴레이션이 꼭 다른 릴레이션일 필요는 없다. 즉 자기 자신의 기본키를 참조할 수도 있다. 


### 정리
![](https://velog.velcdn.com/images/chocochip/post/c7be5ab3-fb5c-43f0-a6a9-de3de44a6da7/image.jpeg)