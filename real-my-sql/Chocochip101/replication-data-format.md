바이너리 로그에 이벤트가 어떤 포맷으로 기록되는지는 복제가 처리되는 과정에도 영향을 주는데, 레플리카 서버가 소스 서버의 바이너리 로그 이벤트를 내부적으로 가공하지 않고 가져온 그대로 실행해 자신의 데이터에 적용하므로 복제에서 어떤 바이너리 로그 포맷을 사용하느냐는 중요한 부분이다. MySQL에서는 실행된 SQL문을 바이너리 로그에 기록하는 Statement 방식과 변경된 데이터 자체를 기록하는 Row 방식으로 두 종류의 바이너리 로그 포맷을 제공하며, 사용자는 binlog_format 시스템 변수를 통해 이 두 가지 종류 중 하나로 설정하거나 혹은 혼합된 형태로 사용하도록 설정할 수 있다. 두 방식이 서로 매우 상이한 특징들을 가지므로 하나씩 나눠서 살펴보자.

# Statement 기반 바이너리 로그 포맷 
Statement 기반 바이너리 로그 포맷은 MySQL에 바이너리 로그가 처음 도입됐을 때부터 존재해왔던 포맷으로, 앞서 언급한 바와 같이 변경 이벤트에 대해 **이벤트를 발생시킨 SQL문을 바이너리 로그에 기록하는 방식**이다. 다음은 바이너리 로그를 사람이 읽을 수 있는 형태로 변환하는 툴인 mysqlbinlog로 확 인한 Statement 포맷 형식의 바이너리 로그의 내용이다. 

Statement 포맷 형식의 바이너리 로그는 실행된 SQL문이 그대로 바이너리 로그에 저장돼 있는 것을 확인할 수 있다. **하나의 SQL문은 여러 개의 데이터를 수정할 수 있는데, 이 경우 Statement 포맷에서는 바이너리 로그에 SQL문 하나만 기록**된다. 이렇게 되면 바이너리 로그 파일의 용량이 작아지므로 사용자 입장에서는 저장 공간에 대한 부담을 덜 수 있으며, 원격으로 바이너리 로그를 백업하거나 혹은 원격에 위치한 레플리카 서버와 복제할 때도 좀 더 빠르게 처리될 수 있다. 바이너리 로그는 변경 내역이 전부 저장되는 파일이므로 감사 등의 목적으로도 활용될 수 있는데, Statement 포맷을 사용하면 손쉽게 SQL문들을 확인할 수 있으므로 감사에 더 용이하다고 할 수 있다. 

## 단점
### 1. 비확정적인 쿼리

비확정적(Non-Deterministic)으로 처리될 수 있는 쿼리가 실행된 경우 Statement 포맷에서는 복제 시 소스 서버와 레플리카 서버 간에 데이터가 달라질 수 있다는 점이다. 

다음은 Statement 포맷 기반 복제에서 소스 서버와 레플리카 서버 간의 데이터 일 관성을 해칠 수 있는 비확정적 쿼리 유형의 몇 가지 예다. 
- DELETE/UPDATE 쿼리에서 ORDER BY 절 없이 LIMIT 사용 
- SELECT FOR UPDATE 및 SELECT FOR SHARE 쿼리에서 NOWAIT나 SKIP LOCKED 옵션 사용
- LOAD_FILE(), UUID(). UUID_SHORT(), USER(), FOUND_ROWS(), RAND(), VERSION() 등과 같은 함수를 사용하는 쿼리 
- 동일한 파라미터 값을 입력하더라도 결괏값이 달라질 수 있는 사용자 정의 함수(User-defined Function)나 스토어드 프로시저(Stored Procedure)를 사용하는 쿼리

### 2. 락
Row 포맷으로 복제될 때보다 데이터에 락을 더 많이 건다는 점인데, INSERT INTO SELECT 구문이 대표적인 경우이고 그 외에 데이터 검색 조건으로 주어진 칼럼에 대한 적절한 인덱스가 테이블에 존재하지 않아 풀 테이블 스캔을 유발하는 UPDATE 쿼리가 실행된 경우 등이 있다. 이 같은 쿼리들은 쿼리를 실행할 때 불필요하게 많은 데이터에 대해 오랜 시간 동안 락을 걸 가능성이 있는데, 소스 서버에서는 당연히 유입된 쿼리 형태 그대로 실행되므로 어쩔 수 없지만 복제로 넘어갈 때는 복제 데이터 포맷에 따라 처리 방식이 달라진다. Statement 포맷이 아닌 Row 포맷이 사용된 경우 레플리카 서버에는 변경된 데이터 자체가 넘어가서 적용되므로 소스 서버에서처럼 쿼리를 실행해 처리하는 형태가 아니기 때문에 동일한 데이터 변경일지라도 락을 더 적게 점유하고 처리 속도도 훨씬 빠르다고 할 수 있다.

### 3. REPEATABLE-READ
Statement 기반 바이너리 로그 포맷은 사용할 때 트랜잭션 격리 수준이 반드시 "REPEATABLE-READ" 이상이어야 한다. 그 이하의 방식에서는 하나의 트랜잭션 내에서도 각 쿼리가 실행되는 시점마다 데이터 스냅숏이 달라질 수 있는데, 이로 인해 복제 시 소스 서버와 레플리카 서버의 데이터가 일치하지 않게 될 수 있으므로 Statement 포맷 사용이 허용되지 않는다.

# Row 기반 바이너리 로그 포맷 
Row 기반 바이너리 로그 포맷은 **MySQL 서버에서 데이터 변경이 발생했을 때 변경된 값 자체가 바이너리 로그에 기록되는 방식**이다. Row 기반 바이너리 로그 포맷은 Statement 기반 바이너리 로그 포맷보다 더 나중에 구현된 방식이지만 어떤 형태의 쿼리가 실행 됐든 간에 복제 시 **소스 서버와 레플리카 서버의 데이터를 일관되게 하는 가장 안전한 방식**이다. 또한 MySQL 5.7.7 버전부터는 바이너리 로그의 기본 포맷으로 지정되기도 했다. Row 포맷에서는 소스 서버에서 실행된 쿼리가 UUID(), USER() 등과 같은 비확정적 함수를 사용했다 하더라도 레플리카 서버에서 똑같이 이 함수가 다시 실행되는 것이 아니라 함수의 결괏값을 전달받아 처리되므로 이 같은 경우에 있어서도 안전하게 복제가 가능하다. 또한 앞에서 언급한 것처럼 Row 포맷 에서는 MySQL 서버에서 다음과 같은 쿼리들이 실행되는 경우 Statement 포맷보다 락이 최소화되어 처리된다. 그리고 레플리카 서버에서도 쿼리가 실행되는 것이 아니라 변경된 데이터가 바로 적용되므 로 어떤 변경 이벤트건 더 적은 락을 점유하며 처리된다.

- INSERT SELECT 
- INSERT with AUTO_INCREMENT 
- 적절한 인덱스가 없어 풀스캔으로 처리되는 UPDATE/DELETE 

변경된 데이터가 그대로 바이너리 로그에 기록된다는 것은 가장 큰 장점이자 단점이 될 수 있다. 

만약 MySQL 서버에서 실행된 쿼리가 굉장히 많은 데이터를 변경한 경우에는 변경된 데이터가 전부 기록되므로 바이너리 로그 파일 크기가 단시간에 매우 커질 수 있다. 또한 변경된 데이터 수가 적더라도 BLOB 형태의 큰 값이 새로 저장되거나 변경되는 경우에는 마찬가지로 파일 크기가 많이 커질 수 있음을 유의 해야 한다. 레플리카 서버에 소스 서버의 이벤트들이 Row 포맷으로 넘어오면 사용자는 소스 서버로부터 어떤 쿼리들이 넘어왔고 현재 실행 중인 쿼리가 어떤 것인지 레플리카 서버의 MySQL에서 바로 육안으로 확인할 수가 없다. 따라서 실행된 변경 내역을 SQL문 형태로 확인하려면 레플리카 서버의 릴레이 로그나 바이너리 로그(log_slave_updates 옵션이 활성화돼 있는 경우)를 mysqlbinlog 프로그램을 사용해 변환해야 한다. 이때 "-v(--verbose)" 옵션을 반드시 사용해야 하는데, 이 옵션을 지정하면 mysqlbinlog 는 변경된 데이터를 유사 SQL(pseudo-SQL) 형태로 변환해서 보여준다. 만약 소스 서버에서 실행된 SQL문을 그대로 보고 싶다면 소스 서버에서 binlog_rows_query_log_events 시스템 변수를 활성화한 후 mysqlbinlog를 사용할 때 "-vv(--verbose --verbose)" 옵션을 명시하면 된다. 이 경우 변환된 결과 파일의 Rows_query 섹션에서 실제 실행된 원본 쿼리를 확인할 수 있다. 이 두 옵션 모두 변환된 결과 파일에 Base64 문자열로 인코딩된 변경 데이터를 포함하는데, 만약 이 내용을 결과 파일에서 제외하고 싶다면 "--base64-outpit=DECODE-ROWS" 옵션을 함께 사용하면 된다. 

Row 포맷은 모든 트랜잭션 격리 수준에서 사용 가능하며, MySQL 서버의 바이너리 로그 포맷이 Row 포맷으로 설정돼 있다 하더라도 사용자 계정 생성과 권한 부여 및 회수, 그리고 테이블과 뷰, 트리거 생성 등과 같은 DDL문은 전부 Statement 포맷 형태로 바이너리 로그에 기록된다. 

# Mixed 포맷
사용자는 MySQL 서버가 두 가지 바이너리 로그 포맷을 혼합해서 사용하도록 설정할 수 있는데, MySQL 서버의 binlog_foramt 시스템 변수를 MIXED 값으로 지정하면 된다. MySQL 서버는 MIXED 포맷으로 설정되면 **기본적으로는 Statement 포맷을 사용하며, 실행된 쿼리와 스토리지 엔진의 종류에 따라 필요 시 자동으로 Row 포맷으로 전환해서 로그에 기록**한다. 쿼리의 경우 대부분 Statement 포맷으 로 기록될 가능성이 높은데, 만약 실행된 쿼리가 Statement 포맷으로 기록되어 복제됐을 때 문제가 될 가능성이 있는 안전하지 못한 쿼리 형태라면 Row 포맷으로 변환되어 기록된다. 여기서 복제에 안전하 지 못한 쿼리는 앞에서 언급한 비확정적으로 처리될 수 있는 쿼리 유형을 가리킨다. MySQL 서버의 스토리지 엔진별로 사용 가능한 바이너리 포맷이 다른데, 이에 따라 기록되는 포맷이 달라질 수도 있다.

- InnoDB: Row 포맷 지원, Statement 포맷 지원

MIX 방식을 사용하면 Statement 포맷과 Row 포맷의 장점을 취해서 사용하는 것으로 생각할 수 있지만 MySQL 서버가 내부적으로 설정된 기준과 기술적인 측면을 고려해 자동으로 두 포맷을 번갈아 사용하는 것이므로 실제 사용자가 예상했던 것과 다르게 처리될 수도 있다. 적합한 방식을 선정하자.

# Row 포맷의 용량 최적화
바이너리 포맷에 비교하여 Row 포맷은 로그 파일의 용량이 크다. MySQL에서는 용량 문제를 보완하도록 Row 포맷을 사용할 때 로그 파일의 용량을 줄일 수 있는 두 가지 방식을 제공한다.

## 바이너리 로그 Row 이미지
Row 포맷은 변경된 데이터들이 전부 저장되기 때문에 많은 저장공간과 네트워크 트래픽을 유발할 가능성이 있다. Mixed 포맷으로 설정되도 이 가능성은 남아있기에 MySQL에서는 Row 포맷의 바이너리 로그 파일 용량을 최소화하기 위해 저장되는 변경 데이터의 칼럼 구성을 제어하는 binlog_row_image라는 시스템 변수를 제공한다.

Row 포맷은 바이너리 로그에서 변경 전 레코드와 변경 후 레코드가 함께 저장되는데, binlog_row_image 시스템 변수는 각 변경 전후 레코드들에 대해 테이블의 어떤 칼럼들을 기록할 것인지를 결정한다. 사용자는 binlog_row_image 시스템 변수 셋 중 하나를 선택할 수 있으며, default는 full이다.

- full: 특정 칼럼에서만의 변경 여부와 관계없이 변경이 발생한 레코드의 모든 칼럼들의 겂을 바이너리 로그에 기록하는 방식이다.
- minimal: 변경 데이터에 대해 꼭 필요한 칼럼들의 값만 바이너리 로그에 기록한다.
- noblob: full 옵션을 설정한 것과 동일하게 작동하지만 레코드의 BLOB이나 TEXT 칼럼에 대해 변경이 발생하지 않은 경우 해당 칼럼들은 바이너리 로그 파일에 기록하지 않는다.

|이벤트 종류|변경 전 레코드|변경 후 레코드|
|---|---|---|
|INSERT|(없음)|INSERT의 모든 칼럼과 Auto-Increment 값|
|UPDATE|PKE|INSERT의 모든 칼럼|
|DELETE|PKE|(없음)|

## 바이너리 로그 트랜잭션 압축
일반적으로 MySQL 서버의 바이너리 로그는 안정적인 복제를 위해 일정 기간 동안 보관되도록 설정하며, 또한 시점 복구(Point-In-Time Recovery)를 고려하는 경우에는 원격 스토리지 서버에 바이너리 로그들을 백업해두기도 한다. 따라서 MySQL 서버에서 생성되는 바이너리 로그 파일의 양이 많은 경우에는 디스크 저장 공간은 물론 네트워크 대역폭을 많이 소비하게 된다. 

바이너리 로그 포맷을 Row로 사용 중인 상태에서 Row 이미지를 조정했다고 하더라도 유입되는 DML 퀴리의 양이 많은 MySQL 서버에서는 바이너리 로그 파일의 크기가 커질 수밖에 없다. 이 경우 사용자는 디스크 저장 공간과 네트워크 대역폭 사용량 절약을 위해 바이너리 로그 보관 주기를 더 짧게 설정 하고, 원격 스토리지 서버에 저장할 때는 별도의 툴을 사용해 바이너리 로그 파일들을 압축한 뒤 전송할 수 있다. 그러나 원격의 레플리카 서버로 바이너리 로그를 전송함에 따라 소비되는 네트워크 대역폭 사용량은 사용자가 줄일 수 없으며, 또한 안정적인 복제와 신규 레플리카 서버 구축 등을 위해서는 바 이너리 로그 보관 주기를 짧게 설정하는 것도 한계가 있다. MySQL 8.0.20 버전에서 Row 포맷으로 기록되는 트랜잭션에 대해 트랜잭션에서 변경한 데이터를 압축해서 바이너리 로그에 기록할 수 있게하는 기능이 도입됐다. 따라서 사용자는 기존과 동일한 바이너리 로그 보관 주기를 유지하면서 이전보다 디스크 공간을 절약할 수 있게 됐으며, 복제로 인해 소비되 는 네트워크 대역폭 사용량도 줄일 수 있게 됐다. 

![](https://velog.velcdn.com/images/chocochip/post/cffa86cb-e656-4de9-8496-cf91a991f284/image.png)


소스 서버의 트랜잭션 데이터가 압축되 어 레플리카 서버로 복제되는 과정을 보여준다.

MySQL 서버에서 바이너리 로그 트랜잭션 압축 기능이 활성화돼 있으면 **트랜잭션에서 변경한 데이터들을 zstd 알고리즘을 사용해 압축한 뒤 Transaction_ payload_event라는 하나의 이벤트로 바이너리 로그에 기록**한다. 압축된 트랜잭션 데이터는 레플리카 서버로 복제될 때도 압축된 상태를 유지하며, 레플리카 서버의 레플리케이션 I/O 스레드도 압축된 상태 그대로 릴레이 로그에 기록한다. 따라서 소 스 서버와 레플리카 서버 모두에서 디스크 저장 공간이 절약될 수 있으며, **네트워크 대역폭 사용량도 줄어든다**. 

그러나 한 가지 주의해야 할 점은 소스 서버에서 바이너리 로그 압축이 적용된 경우 레플리카 서버도 반드시 바이너리 로그 압축을 지원하는 MySQL 8.0.20 이상의 버전을 사용해야 한다는 것이다. MySQL 8.0.20 미만의 버전을 사용하는 레플리카 서버는 바이너리 로그 압축이 적용된 소스 서버에 대해 복제가 불가능하기 때문이다. 사용자는 binlog_transaction_compression 시스템 변수를 통해 압축 기능을 활성화할 수 있으며, binlog_transaction_compression_level_zstd 시스템 변수를 통해 압축 시 사용될 zstd 알고리즘 레벨을 설정할 수 있다. 

- binlog_transaction_compression: ON(1) 또는 OFF(0)로 설정 가능하며, 기본값은 OFF다. 
- binlog_transaction_compression_level_zstd: 압축 레벨은 1부터 22까지 값을 지정할 수 있으며, 기본값은 3이다. 압축 레벨이 높을수록 압축률이 증가해 디스크 공간이나 네트워크 대역폭을 더 절약할 수 있다는 장점이 있지만 CPU와 메모리 사용량이 늘어나고 처리 시간이 증 가할 수 있다. 또한 압축 레벨이 높다고 해서 반드시 압축률이 좋아지는 것은 아니다. 

두 시스템 변수는 세션별로도 설정할 수 있어서 글로벌하게 압축 기능을 적용하지 않고 세션에서 트랜잭션별로 선택적으로 압축 기능을 적용할 수도 있다. 즉, 바이너리 로그에 압축된 트랜잭션 데이터와 압축되지 않은 트랜잭션 데이터가 혼합되어 존재할 수 있는 것인데, 이러한 경우에도 복제 등에서 문제 없이 처리가 가능하다. 바이너리 로그 트랜잭션 압축은 모든 경우에 대해 압축을 적용하지 않으며, 대표적으로 다음과 같은 이 벤트 타입들은 압축 기능이 활성화돼 있다 하더라도 항상 바이너리 로그에 압축되지 않은 상태로 기록 된다. 

- GTID 설정 관련 이벤트 
- 그룹 복제에서 발생하는 View Change 이벤트 또는 소스 서버에서 레플리카 서버에 살아있음을 알리는 Heartbeat 이벤트와 같은 제어 이벤트(Heartbeat 이벤트는 바이너리 로그에 실제로 기록되지는 않는다.) 
- 복제 실패 및 소스 서버와 레플리카 서버 간 데이터 불일치를 발생시킬 수 있는 Incident 타입의 이벤트 
- 트랜잭션을 지원하지 않는 스토리지 엔진에 대한 이벤트 및 그러한 이벤트를 포함하고 있는 트랜잭션 이벤트 
- Statement 포맷으로 기록되는 트랜잭션 이벤트(바이너리 로그 포맷이 MIXED로 설정돼 있는 경우에 해당한다고 볼 수 있다. 바이너리 로그 트랜잭션 압축 기능은 Row 포맷으로 기록되는 이벤트들에만 적용된다.) 

압축된 트랜잭션 데이터는 트랜잭션의 개별 이벤트들의 내용이 어떤 것인지 실제로 확인이 필요할 때 압축이 해제되는데. 다음과 같은 경우가 여기에 해당한다. 

- 레플리카 서버에서 레플리케이션 SQL 스레드에 의해 복제된 트랜잭션이 적용될 때 
- mysqlbinlog를 사용해 트랜잭션을 재실행할 때 
- SHOW BINLOG EVENTS 혹은 SHOW RELAYLOG EVENTS 구문이 사용될 때 

사용자는 mysqlbinlog를 통해 압축된 트랜잭션 데이터에 대해 압축된 크기와 압축되지 않은 크기를 나 타내는 설명과 사용된 압축 알고리즘을 확인할 수 있다. 이때"--verbose(-v)" 옵션을 반드시 명시해야 한다. 


- "Start of compressed events!" 메시지: 압축 시작
- "End of compressed events!" 메시지: 압축 종료
- 두 메시지에 둘러싸인 부분: 압축 내용
- payload_size: 압축된 데이터의 크기
- compression_type: 사용된 압축 알고리즘
- uncompressed_size: 압축되지 않았을 때의 데이터 크기

사용자는 Performance 스키마를 통해 압축된 트랜잭션들의 통계 정보와 압축 성능을 확인할 수 있다. Performance 스키마의 binary_log_transaction_compression_stats 테이블에 바이너리 로그와 릴레이 로그에 기록된 트랜잭션들에 대한 압축 통계 정보가 저장된다. 따라서 일반적으로 소스 서버에서는 해 당 테이블에 바이너리 로그에 대한 통계 정보만 표시되고 레플리카 서버에는 릴레이 로그에 대한 통계 정보가 표시되는데, 만약 레플리카 서버에서 바이너리 로그 및 log_slave_updates 설정이 활성화돼 있으면 릴레이 로그와 더불어 바이너리 로그에 대한 통계 정보도 함께 표시된다. 


테이블에는 로그 파일 종류와 압축 여부별로 통계 정보가 나눠져 있으며, 테이블의 LOG_TYPE 칼럼을 통 해 로그 파일 종류를 확인할 수 있고 COMPRESSION_TYPE 칼럼을 통해서는 압축 여부 및 압축에 사용된 알고리즘을 확인할 수 있다. 또한 각 통계 정보에는 통계 정보가 수집되기 시작한 시점부터 지금까지 기 록된 트랜잭션 수와 총 크기, 전체적인 압축률 및 첫 번째로 기록된 트랜잭션과 마지막으로 기록된 트 랜잭션에 대한 부가적인 정보들이 포함돼 있다. 테이블에 저장되는 통계 정보는 일반적으로 MySQL 서버가 시작되면 자동으로 수집되며, 사용자는 다 음과 같이 TRUNCATE 구문을 사용해 MySQL 서버가 구동 중인 상태에서 테이블 데이터를 초기화할 수도 있다. 이 경우 초기화된 시점부터 다시 통계 정보가 수집된다. 참고 binary_log_transacticn_compression_stats 테이블의 예제 데이터에서 릴레이 로그에 대한 통계 정보들 9FIRST_TRANSACTION_TIMESTA 칼럼과 LAST_TRANSACTION_TIMESTAMP 칼럼에 값이 먼 미래 값으로 잘못 표기 돼 있는 것을 알 수 있는데, 현재 MySQL 버그 페이지 에도 해당 내용이 제보된 상태이며 MySQL 8.0.25 버전에서도 아직 수정되지 않은 것으로 보인다. 압축 성능과 관련해서 트랜잭션의 압축 및 압축 해제에 소요된 시간도 Performance 스키마를 통해 확 인할 수 있는데, 이를 위해서는 Performance 스키마가 해당 정보들을 수집하도록 다음의 UPDATE 문을 사용해 Performance 스키마의 설정을 변경해야 한다. 주의 Performance 스키마에서 수집되는 정보들이 늘어나면 MySQL 서버에 부하를 줄 수 있으므로 실제로 서비스 에서 사용 중인 MySQL 서버에서 Performance 스키마 설정을 바로 변경하기보다는 별도로 구축한 테스트 환경에서 Performance 스키마 설정 변경에 따른 영향도를 확인한 후 서비스 MySQL 서버의 설정을 변경하는 것이 좋다. Performance 스키마 설정을 변경한 후 다음 쿼리를 실행하면 MySQL 서버가 트랜잭션을 압축하고 압축을 해제하는 데 소요한 시간에 대한 통계 정보를 확인할 수 있다.

한 가지 유의할 점은 위 소요 시간 통계 정보는 앞서 Performance 스키마 설정을 변경한 이후부터 수 집된 데이터이므로 압축 기능이 적용된 직후부터 바로 통계 정보가 수집되도록 설정하고 싶다면 압축 기능을 적용하기 전에 Performance 스키마 설정을 미리 변경해둬야 한다는 것이다. 압축 기능 활성화 옵션이 MySQL 설정 파일에 명시돼 있고 통계 정보 또한 MySQL 서버가 시작된 직후부터 수집되도록 설정하고 싶다면 MySQL 설정 파일에 다음과 같이 옵션을 명시해야 한다. Performance 스키마 설정 에 대한 자세한 내용은 18.3절 'Performance 스키마 설정'을 살펴보자.

그림 16.8은 바이너리 로그 트랜잭션 압축 기능을 활성화했을 때 바이너리 로그 파일의 크기가 어느 정 도 줄어드는지 간단하게 테스트해본 결과다. 테스트에는 sysbench 툴이 사용됐으며, 50만 건의 데이터 를 저장하는 경우(그림에서 BULK INSERT)와 OLPT 성격의 Read-Write 쿼리들이 실행되는 경우 (그림에서 OLTP READ WRITE) 별로 압축 기능 사용 여부에 따른 바이너리 로그 파일의 크기를 확인 했다. 압축 레벨은 기본으로 설정되는 값(3)을 그대로 사용했다. 그림 16.8에서 보다시피 테스트 결과, 두 경우 모두 압축을 적용한 후 약 50% 정도로 바이너리 로그 파 일의 크기가 줄어든 것을 확인할 수 있었다. 테스트는 특정 경우에 대해서만 진행된 것이므로 MySQL 서버에서 사용되는 쿼리 패턴에 따라 압축률은 달라질 수 있음을 참고하자. 그림 16.9는 앞서 테스트한 두 경우에 대해 압축 레벨을 다르게 설정해서 테스트한 결과다. 그림을 보 면 알 수 있듯이 binlog_transaction_compression_level_zstd 시스템 변수의 값을 변경했는데도 두 경우 모두 압축 레벨별로 바이너리 로그 파일의 크기가 거의 동일했다. 테스트 결과만 봤을 때 MySQL 서버 에서 압축 레벨이 정상적으로 잘 적용되어 작동하는지는 의문스러운 부분이 있지만 일단 현재로서는 압축 레벨은 기본으로 설정되는 값(Level=3)을 그대로 사용해도 무방해 보인다.

바이너리 로그 트랜잭션 압축 기능을 사용하면 압축 처리로 인해 MySQL 내부적으로 오버헤드가 존재 하며, CPU와 메모리 등의 서버 자원을 더 소모할 수 있다. 평균 CPU 사용률은 큰 차이가 없지만 소요 시간이 꽤 차이가 난다. 이처럼 압축 기능을 사용하면 오버헤드로 인해 압축 기능을 사용하지 않을 때보다 쿼리 처리가 지연될 수 있고 서버의 자원도 더 소모하게 되므로 압축 기능을 사용하고자 할 때는 현재 MySQL 서버의 리소 스 사용률 현황과 서비스 요건을 충족시키는 쿼리 응답 속도 등을 파악한 후 별도로 구축한 테스트 환 경에서 성능을 확인해 압축 기능의 사용 여부를 결정하는 것이 좋다.