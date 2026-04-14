## 트랜잭션과 격리 수준 (Transaction And Isolation Level)

 - 실제 트랜잭션은 JDBC, JPA(Hibernate), R2DBC 등 하위 기술이 처리

```text
Application Code
↓
@Transactional (선언적 트랜잭션)
↓
TransactionInterceptor (AOP 프록시)
↓
PlatformTransactionManager (추상화)
↓
┌──────────────┬──────────────┬──────────────┐
│ DataSource   │ JpaTransaction│ R2dbc        │
│ TxManager    │ Manager       │ TxManager    │
│ (JDBC)       │ (Hibernate)   │ (Reactive)   │
└──────────────┴──────────────┴──────────────┘
```
- `PlatformTransactionManager`는 Spring Framework의 `spring-tx` 모듈에 정의된 인터페이스이며, `getTransaction(), commit(), rollback()` 3개 메서드만 가지고 있다.

**선언적 트랜잭션 — `@Transactional`의 동작 원리**

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(Long fromId, Long toId, int amount) {
        accountRepository.debit(fromId, amount);   // A 출금
        accountRepository.credit(toId, amount);     // B 입금
        // 예외 발생 시 → 둘 다 롤백
    }
}
```
- Spring AOP가 프록시 객체를 만들어서 메서드 호출 전후에 트랜잭션을 자동으로 관리

**핵심 동작 흐름**
1. 외부에서 transferService.transfer() 호출
2. 프록시가 가로채서 `TransactionManager.getTransaction()` 호출 → 커넥션 획득, autocommit = false 설정
3. 실제 transfer() 메서드 실행
4. 정상 완료 → commit() / 예외 발생 → rollback()

**반드시 알아야 할 함정: 프록시 기반의 한계, 같은 클래스 내부에서의 호출은 @Transactional이 무시된다.**

```java
@Service
public class OrderService {

    public void process() {
        this.saveOrder();  // ← 프록시를 거치지 않음. 트랜잭션 미적용!
    }

    @Transactional
    public void saveOrder() {
        // ...
    }
}
```
- `Why?`: this.saveOrder()는 프록시 객체가 아닌 실제 객체의 메서드를 직접 호출하기 때문 (Spring AOP가 JDK Dynamic Proxy 또는 CGLIB 프록시로 동작하는 구조적 한계)
- `Solution`: `트랜잭션이 필요한 메서드를 별도 빈(클래스)으로 분리`하거나, `ApplicationContext에서 자기 자신의 프록시를 주입받는 방식`을 쓴다.

**###`ApplicationContext에서 자기 자신의 프록시를 주입받는 방식` 예제 ###**
```java
@Service
public class ExampleService {
    
    /* 자기 자신이지만 proxy 참조 기대 */
    @Autowired
    private ExampleService self;
    
    public void generalMethod () {
        self.transactionalMethod(); // proxy를 통해 호출
    }
    
    @Transactional
    public void transactionalMethod () {
        // db 작업...
    }
}
```

### 전파(Propagation) 
- 트랜잭션 안에서 또 트랜잭션을 만난다면? `@Transactional(propagation = ...)` 옵션으로 제어
    - `REQUIRED (기본값)`: 이미 트랜잭션이 있으면 참여하고, 없으면 새로 만든다.
    - `REQUIRES_NEW`: 기존 트랜잭션을 일시 중단하고 완전히 새로운 트랜잭션을 시작한다 (`주문은 실패해도 감사 로그는 남겨야 하는 요구사항에 사용`)
                  <br/> 단, 새 커넥션을 추가로 점유하므로 커넥션 풀 고갈 리스크 존재
    - `NESTED`: 세이브포인트를 찍고, 내부 실패 시 세이브포인트까지만 롤백한다. 
            <br/> JPA/Hibernate에서는 지원하지 않고, 순수 JDBC(DataSourceTransactionManager)에서만 동작

```text
<<주의>> 
REQUIRES_NEW와 NESTED를 혼동하는 경우가 많은데, REQUIRES_NEW는 물리적으로 별개의 트랜잭션(별개 커넥션)이고, NESTED는 같은 물리 트랜잭션 안의 세이브 포인트이다.
외부 트랜잭션이 롤백되면 NESTED는 함께 롤백되지만 REQUIRES_NEW는 이미 커밋된 것은 유지된다.
```

**격리 수준(Isolation Level) — 동시성 문제의 핵심**
- `Oracle`: 기본이 READ_COMMITTED
- `MySQL(InnoDB)`: 기본이 REPEATABLE_READ

`@Transactional(isolation = Isolation.READ_COMMITTED)` 처럼 Spring에서 지정할 수 있지만, 이건 Spring이 `Connection.setTransactionIsolation()`을 호출해서 DB에 요청하는 것이지 Spring이 직접 격리를 구현하는 것은 아니다.

InnoDB의 구현 특성상 `MySQL InnoDB`는 `MVCC + Next-Key Lock`으로 `REPEATABLE_READ`에서도 `Phantom Read`를 대부분 방지 
`READ_COMMITTED 환경에서 같은 트랜잭션 내에 같은 쿼리를 두 번 날렸을 때 결과가 달라질 수 있다(Non-Repeatable Read)`


**Spring `@Transactional`의 기본 롤백 규칙**
- RuntimeException, Error → 롤백
- Checked Exception → 롤백하지 않음 (커밋됨)
- IOException 같은 checked exception이 발생해도 트랜잭션은 커밋 된다. <br/>
  명시적으로 `@Transactional(rollbackFor = Exception.class)`를 지정해야 checked exception에서도 롤백 동작

**점검 항목**
- `자기 호출(self-invocation) 함정` — 기존 코드에서 같은 클래스 내에서 @Transactional 메서드를 호출하는 곳이 없는지 점검
- `전파 속성 확인` — REQUIRES_NEW를 쓰는 곳이 있다면 커넥션 풀 사이즈(HikariCP maximumPoolSize)와의 관계를 확인, `중첩 호출 깊이 × 동시 요청 수`가 풀 사이즈를 넘으면 데드락이 발생한다.
- `롤백 규칙` — 프로젝트 전반에서 checked exception을 던지는 서비스 메서드가 있는지, 그 메서드에 rollbackFor가 지정되어 있는지 확인
- `격리 수준` — Oracle 환경이라면 READ_COMMITTED가 기본이므로, 동일 트랜잭션 내 반복 조회가 필요한 로직에서 데이터 변경 가능성을 고려


**트랜잭션 추상화 코드가 DB 레벨까지 내려가는 것인가?**
- 스크립트
```mysql
START TRANSACTION;

-- 각종 DML...

COMMIT;
```

- 최종적으로 DB에 SQL 수준의 명령이 내려간다. 다만 위의 `스크립트` 형태 보다는 JDBC 드라이버를 통한 DB 프로토콜 명령이라고 보는 것이 정확하다.

`@Transactional`이 붙은 메서드가 호출되면 내부적으로 아래와 같이 실행된다.
```text
 @Transactional 메서드 진입
    → TransactionInterceptor (AOP 프록시)
    → PlatformTransactionManager.getTransaction()
    → DataSource.getConnection()       ← HikariCP 등 커넥션 풀에서 커넥션 획득
    → Connection.setAutoCommit(false)  ← 이게 JDBC API 호출
```

- 이 `Connection.setAutoCommit(false)`가 JDBC 드라이버 내부에서 실제 DB로 전달된다
- Oracle JDBC 드라이버의 경우, Oracle은 기본이 auto-commit off 상태이고 명시적 `BEGIN TRANSACTION` 구문이 없고, 첫 DML 실행 시점에 암묵적으로 트랜잭션이 시작된다.
- MySQL이라면 드라이버가 실제로 `SET autocommit=0`을 DB 서버 보낸다.

- 그리고 메서드 실행이 종료된다면 
```text
 메서드 정상 종료
    → TransactionManager.commit()
    → Connection.commit()  ← JDBC API 호출
    → Oracle Net Protocol로 COMMIT 명령 전송
    → Oracle DB 엔진이 redo log flush + 락 해제
```

예외 발생 시
```text
  RuntimeException 발생
    → TransactionManager.rollback()
    → Connection.rollback()        ← JDBC API 호출
    → Oracle Net Protocol로 ROLLBACK 명령 전송
    → Oracle DB 엔진이 undo 적용 + 락 해제
```

추상화 계층을 관통하는 전체 그림
```text
 [ Java 코드 레벨 ]        
 @Transactional
        ↓
 [ Spring 레벨 ]           
 PlatformTransactionManager
        ↓
 [ JDBC 표준 API ]         
 java.sql.Connection (.setAutoCommit(...), .commit(...), .rollback(...))
        ↓
 [ JDBC 드라이버 ]         
 ojdbc (Oracle) / mysql-connector-j (MySQL) → DB 벤더 전용 네트워크 프로토콜로 변환
        ↓
 [ 네트워크 ]              
 TCP/IP 소켓 통신
        ↓
 [ DB 서버 레벨 ]          
 COMMIT / ROLLBACK 명령 수신 → 트랜잭션 로그(redo/undo) 처리 → 락 관리, 버퍼 플러시
```

- Spring의 `@Transactional`은 결국 `java.sql.Connection`의 3개 메서드(`setAutoCommit, commit, rollback`)로 수렴
- JDBC 드라이버가 이 호출을 벤더별 프로토콜로 변환한다.
  - Oracle이면 `Oracle Net(TNS)`,  MySQL이면 `MySQL Client/Server Protocol`

```text
"Spring이 BEGIN TRANSACTION SQL을 날리는 거 아닌가?" 라고 생각할 수 있는데, 아닙니다.
JDBC 스펙에는 BEGIN 구문을 직접 보내는 API가 없습니다.
setAutoCommit(false) 호출이 "이 커넥션에서 이후 실행되는 SQL은 명시적 commit/rollback 전까지 하나의 트랜잭션으로 묶어라"는 의미이고, 
이걸 DB 벤더별 드라이버가 각자의 방식으로 처리합니다.
```

## 더 알아보기: 명시적 트랜잭션 관리 

1. `java.sql.Connection` 사용 
2. `org.springframework.transaction.PlatformTransactionManager`사용


```java

import javax.sql.DataSource;
import java.sql.Connection;

@Service
@RequiredArgsConstructor
class TestClass {

    private final DataSource dataSource;
    private final PlatformTransactionManager transactionManager;

    @Test
    void test1() {
        /* 명시적으로 관리하면 connection 단위로 트랜잭션이 형성되므로 메서드 외부에서 connection을 인수로 주입 받아 사용하는 것이 올바르다... */
        Connection connection = null;
        try {
            /* connection 획득 */
            connection = dataSource.getConnection();
            /* auto commit을 끈다... MySQL 이라면 이때 `SET autocommit=0` 쿼리가 나간다. */
            connection.setAutoCommit(false);

            /* 각종 DB 작업... */

            connection.commit();
        } catch (Exception e) {
            if (Objects.nonNull(connection)) connection.rollback();
        } finally {
            if (Objects.nonNull(connection)) connection.close();
        }
    }

    /* PlatformTransactionManager 사용 예제 */

    void test2() throws SQLException {
        /* 트랜잭션 정의 (기본 설정 사용) */
        var transactionStatus = transactionManager.getTransaction(
                new DefaultTransactionDefinition()
        );

        try {
            /* 각종 DB 작업... */

            transactionManager.commit(transactionStatus);
        } catch (Exception e) {
            transactionManager.rollback(transactionStatus);
        }
    }
}
```

