---
title: Spring @Transactional(readOnly = true) 정말 읽기 전용일까?
description:
date: 2025-06-18 00:00:00+0000
math: true
categories:
  - Spring
tags:
  - Spring
  - Transaction
  - readOnly
---

스프링에서는 `@Transactional(readOnly = true)`를 통해 "읽기 전용 트랜잭션"을 설정할 수 있습니다.<br/>
하지만 실제로 이 설정이 모든 쓰기 작업을 막아줄까요?

다음 코드의 테스트 결과는 실패였고, 이유는 예외가 발생하지 않았기 때문이었습니다.

```java
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    void readOnlyUpdate() {
        assertThrows(Exception.class, () -> userService.signUpUser());
    }
}
```

```java
@Service
@RequiredArgsConstructor
class UserService {

    private final UserRepository userRepository;

    @Transactional(readOnly = true)
    public void allUpgradeLevel() {
        List<User> users = userRepository.findAll();
        for (User user : users) {
            user.upgradeLevel();        // 상태 변경
            userRepository.save(user);  // 쓰기 작업
        }
    }
}
```

왜 예외가 발생하지 않았을까요?

## 스프링의 `@Transactional` 어노테이션

스프링의 `@Transactional` 어노테이션은 프록시 기반의 AOP(Aspect-Oriented Programming)를 사용하여 트랜잭션을 적용하는 기능입니다.<br/>
메서드 호출을 가로채서 트랜잭션을 시작하고, 정상적으로 끝나면 커밋(commit), 예외가 발생하면 롤백(rollback)하는 방식입니다.

트랜잭션 관련 다양한 속성을 제공합니다.
- propagation 
- isolation 
- readOnly 
- rollbackFor / noRollbackFor 
- timeout

[Spring 공식 문서](https://docs.spring.io/spring-framework/docs/4.3.x/javadoc-api/org/springframework/transaction/annotation/Transactional.html#readOnly--)
에서 read-only 속성을 확인해 보니 다음과 같이 명시하고 있었습니다.

> This just serves as a hint for the actual transaction subsystem;
> it will not necessarily cause failure of write access attempts.
> A transaction manager which cannot interpret the read-only hint will not throw an exception when asked for a read-only
> transaction but rather silently ignore the hint.

쉽게 말해 `readOnly` 속성은 트랜잭션 서브 시스템에 대한 힌트일 뿐, 쓰기 작업을 강제로 차단하지 않는다고 합니다.

## 스프링의 트랜잭션 흐름

스프링은 JDBC 기반 트랜잭션을 처리할 때 `DataSourceTransactionManager`를 사용합니다. <br/>
트랜잭션이 시작 지점인 `doBegin()` 메서드이며, 내부적으로 다음과 같은 과정을 수행니다.

1. 커넥션 획득 
2. `prepareConnectionForTransaction()` 호출 
3. `con.setReadOnly(true)` 호출 (readOnly 힌트 전달)
4. `enforceReadOnly`가 `true`면 `SET TRANSACTION READ ONLY` 실행

```java
protected void prepareTransactionalConnection(Connection con, TransactionDefinition definition) throws SQLException {
    if (isEnforceReadOnly() && definition.isReadOnly()) {
        try (Statement stmt = con.createStatement()) {
            stmt.executeUpdate("SET TRANSACTION READ ONLY");
        }
    }
}
```

특이한 점은 `readOnly` 설정 외에도 `enforceReadOnly` 속성을 확인한다는 점이다.

- `readOnly`만 `true`: 힌트만 전달되며, JDBC 드라이버가 이를 어떻게 처리할지는 알 수 없다.
- `readOnly`와 `enforceReadOnly` 모두 `true`: 실제 SQL 쿼리로 `SET TRANSACTION READ ONLY`가 실행되어 읽기 전용 모드가 강제된다.

`SET TRANSACTION READ ONLY` 설정을 통해 읽기 전용 모드가 강제되면 이후 제가 실행한, 사용자 레벨 업그레이드와 같은 쓰기 작업이 실행되면 예외가 발생합니다.
그렇다면 왜 예외가 발생하지 않은걸까요?

좀 더 자세히 살펴보면 `enforceReadOnly` 필드는 `DataSourceTransactionManage`에서 관리하는 필드입니다.

```java
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager implements ResourceTransactionManager, InitializingBean {

    // ...

    private boolean enforceReadOnly = false;

    // ...

    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        // ...

        try {
            // ...

            Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);

            // ...

            prepareTransactionalConnection(con, definition);

            // ...

        } catch (Throwable ex) {
            // ...
        }
    }
}
```

```java
public abstract class DataSourceUtils {

    // ...

    @Nullable
    public static Integer prepareConnectionForTransaction(Connection con, @Nullable TransactionDefinition definition) throws SQLException {
        // ...
        if (definition != null && definition.isReadOnly()) {
            try {
                // ...
                con.setReadOnly(true);
            } catch (SQLException | RuntimeException ex) {
                // ...
            }
            // ...
        }
        // ...

        return previousIsolationLevel;
    }
    // ...
}
```

하지만 readOnly 속성은 `Connection` 인터페이스를 통해 관리되고 구현체는 JDBC 드라이버에 존재합니다.
그렇다면 구현체를 살펴보도록 합니다.

### H2 Database

저는 테스트 환경에서 H2를 사용하고 있었고, 테스트 실행 시 예외 없이 정상 동작했습니다.
H2는 `Connection` 인터페이스의 구현체로 `JdbcConnection`을 사용합니다.
`JdbcConnection`에서 read-only 속성을 설정하는 `setReadOnly()` 메서드를 살펴보면 다음과 같습니다.

```java

@Override
public void setReadOnly(boolean readOnly) throws SQLException {
    try {
        // ...
        checkClosed();
    } catch (Exception e) {
        throw logAndConvert(e);
    }
}
```

내부적으로 아무런 동작도 하지 않습니다.
따라서 read-only 속성을 true로 설정해도 `isReadOnly()` 의 결과는 항상 false로,  `SET TRANSACTION READ ONLY`가 실행되지 않고 있습니다.
그 결과 쓰기 작업이 가능했던 것입니다.

### MySQL

MySQL로 테스트 해보면 예외가 발생하는 것을확인할 수 있습니다.
MySQL은 `Connection` 인터페이스의 구현체로 `ConnectionImpl`을 사용합니다.
`ConnectionImpl`의 `setReadOnly()` 메서드는 다음과 같습니다.

```java
public class ConnectionImpl implements JdbcConnection, SessionEventListener, Serializable {
    // ...

    @Override
    public void setReadOnly(boolean readOnlyFlag) throws SQLException {
        setReadOnlyInternal(readOnlyFlag);
    }

    @Override
    public void setReadOnlyInternal(boolean readOnlyFlag) throws SQLException {
        synchronized (getConnectionMutex()) {
            // ...

            this.session.execSQL(null, "SET SESSION TRANSACTION " + (readOnlyFlag ? "READ ONLY" : "READ WRITE"), -1, null, false,
                    this.nullStatementResultSetFactory, null, false);
            // ...
            this.readOnly = readOnlyFlag;
        }
    }
}
```

`readOnly` 속성에 따라 직접 쿼리를 실행하고 있는것을 확인할 수 있습니다.

하지만 이상한 점은 `enforceReadOnly` 속성의 설정 없이 예외가 발생한다는 점입니다.

이전에 스프링의 `DataSourceTransactionManager` 에서 read-only 속성뿐만 아니라 `enforceReadOnly` 속성까지 true인 경우에만 읽기 모드를 강제화 했는데 어떻게 된
일일까요?

MySQL 드라이버의 update 메서드를 살펴보면 원인을 찾을 수 있습니다.
MySQL은 `ClientPreparedStatement` 클래스를 통해 쿼리를 실제로 실행합니다.
실제 쿼리를 실행시키는 `executeUpdateInternal()` 메서드를 살펴보면 다음과 같습니다.

```java
public class ClientPreparedStatement extends com.mysql.cj.jdbc.StatementImpl implements JdbcPreparedStatement {
    protected long executeUpdateInternal(String sql, boolean isBatch, boolean returnGeneratedKeys) throws SQLException {
        synchronized (checkClosed().getConnectionMutex()) {
            // ...

            if (locallyScopedConn.isReadOnly(false)) {
                throw SQLError.createSQLException(Messages.getString("Statement.42") + Messages.getString("Statement.43"),
                        MysqlErrorNumbers.SQL_STATE_ILLEGAL_ARGUMENT, getExceptionInterceptor());
            }
            // ...

            return this.updateCount;
        }
    }
}
```

MySQL 드라이버는 업데이트 쿼리 실행 시점에 연결 객체의 read-only 상태를 체크하고, 읽기 전용 상태일 경우 예외를 발생시키고 있습니다.
때문에 H2와 달리 MySQL을 사용할 때는 테스트가 성공할 수 있었던 것입니다.

### 결론: `readOnly = true`의 진짜 의미

`@Transactional(readOnly = true)`는 트랜잭션 서브 시스템에 "읽기 전용일 가능성이 높다."는 힌트를 제공한다.
스프링은 read-only 속성뿐만 아니라 enforceReadOnly를 통해 읽기 모드를 설정할 수 있는 선택지를 제공하고
실제로 쓰기 작업을 강제로 막으려면 enforceReadOnly = true도 함께 설정해야 하며, 이것이 JDBC 드라이버에서 제대로 지원돼야 한다.
H2처럼 이를 무시하는 드라이버도 있으므로, 실제 효과는 DB와 드라이버에 따라 다르다.

### 마무리

개발 중 테스트나 로컬 환경에서 H2를 자주 사용하는데, 이 때문에 readOnly가 기대대로 동작하지 않아 혼란스러울 수 있다.
이 설정이 정말로 보호막이 되려면, 드라이버의 지원 여부까지 확인하는 습관이 필요하다.
읽기 전용 트랜잭션을 올바르게 활용하기 위해서는 단순히 `@Transactional(readOnly = true)`만 붙이는 것으로는 부족하다.
상황에 따라 `enforceReadOnly`와 DB 드라이버의 특성을 고려한 전략이 필요하다.
