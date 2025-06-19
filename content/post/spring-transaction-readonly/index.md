---
title: Spring @Transactional(readOnly = true)는 정말 읽기 전용 트랜잭션일까?
description: Spring 트랜잭션에서 `readOnly = true` 설정이 실제로 읽기 전용 트랜잭션인지, 내부 동작에 대해 알아본다.
date: 2025-06-19 00:00:00+0000
math: true
categories:
  - Spring
tags:
  - Spring
  - Transaction
  - readOnly
---

스프링에서는 `@Transactional(readOnly = true)`를 통해 **읽기 전용 트랜잭션**을 설정할 수 있다.<br/>
실제로 이 설정이 "쓰기 작업"을 막아줄까?

다음은 `@Transactional(readOnly = true)` 설정에서 사용자의 레벨을 수정하고, 저장하는 간단한 코드다.

```java
@Service
@RequiredArgsConstructor
class UserService {

    private final UserRepository userRepository;

    @Transactional(readOnly = true)
    public void allUpgradeLevel() {
        List<User> users = userRepository.findAll();
        for (User user : users) {
            user.upgradeLevel();
            userRepository.save(user);
        }
    }
}
```

그리고 이에 대한 테스트 코드는 다음과 같다.

```java
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @DisplayName("readOnly = true인 트랜잭션에서 쓰기 작업이 발생하면 예외가 발생해야 한다.")
    @Test
    void readOnlyUpdate() {
        assertThrows(Exception.class, () -> userService.allUpgradeLevel());
    }
}
```

테스트의 결과는 실패였고, 이유는 **예외가 발생하지 않았기 때문이다.**<br/>
이번 포스팅에서는 왜 테스트가 실패했고, 예외가 발생하지 않았는지에 대해 알아볼 것이다.

## 스프링의 `@Transactional` 어노테이션

스프링의 `@Transactional` 어노테이션은 프록시 기반의 AOP(Aspect-Oriented Programming)를 활용해 트랜잭션을 적용하는 기능이다.<br/>
메서드 실행 전 트랜잭션을 시작하고, 정상적으로 종료되면 커밋(commit), 예외가 발생하면 롤백(rollback)하는 방식으로 동작한다.

`@Transactional` 어노테이션은 트랜잭션 관련 다양한 속성을 제공한다.

- propagation: 트랜잭션 전파 방식
- isolation: 트랜잭션 격리 수준
- timeout: 트랜잭션 제한 시간
- rollbackFor / noRollbackFor: 롤백 예외 지정
- readOnly: 읽기 전용 트랜잭션 여부

### `readOnly` 속성

`@Transactional`의 여러 속성 중 `readOnly`를 `true`로 설정하면, 일반적으로 **읽기 전용 트랜잭션**이 실행된다고 알고 있다.<br/>
하지만 실제로는 이 설정이 **쓰기 작업을 강제적으로 막아주지는 않는다.**

[Spring 공식 문서](https://docs.spring.io/spring-framework/docs/4.3.x/javadoc-api/org/springframework/transaction/annotation/Transactional.html#readOnly--)
에서도 다음과 같이 명시하고 있다.

> This just serves as a hint for the actual transaction subsystem;
> it will not necessarily cause failure of write access attempts.
> A transaction manager which cannot interpret the read-only hint will not throw an exception when asked for a read-only
> transaction but rather silently ignore the hint.

즉, `readOnly = true`는 단지 **트랜잭션 서브시스템에 전달되는 힌트**일 뿐이며, 실제로 쓰기 작업을 막을지 여부는 **트랜잭션 매니저와 JDBC 드라이버의 구현에 달려 있다.**

### 트랜잭션 내부 동작

스프링은 `DataSourceTransactionManager`를 통해 JDBC 기반 트랜잭션을 관리한다.<br/>
트랜잭션은 `doBegin()` 메서드에서 시작되며, `readOnly` 속성은 다음 과정을 거쳐 처리된다.

1. 커넥션 획득
2. `DataSourceUtils.prepareConnectionForTransaction()` 메서드 호출
    - `Connection.setReadOnly(true)` 메서드를 통해 JDBC 드라이버에 힌트 제공
3. `prepareTransactionalConnection()` 메서드 호출
    - 읽기 전용 트랜잭션 설정

아래는 `prepareTransactionalConnection()`의 내부 구현 코드다.

```java
protected void prepareTransactionalConnection(Connection con, TransactionDefinition definition) throws SQLException {
    if (isEnforceReadOnly() && definition.isReadOnly()) {
        try (Statement stmt = con.createStatement()) {
            stmt.executeUpdate("SET TRANSACTION READ ONLY");
        }
    }
}
```

여기서 주목할 점은 단순히 `readOnly = true` 설정만으로는 `SET TRANSACTION READ ONLY` 쿼리가 실행되지 않는다는 것이다.<br/>
해당 쿼리는 `readOnly`와 함께 `enforceReadOnly`가 **`true`로 설정되어 있어야만** 실행되며, 이 경우에만 **DB 수준에서 읽기 전용 트랜잭션이 실제로 강제**된다.

- `readOnly = true`: JDBC 드라이버에 힌트를 주는 역할
- `enforceReadOnly = true`: 실제로 `SET TRANSACTION READ ONLY` 쿼리를 실행하여 DB 레벨에서 강제 적용

### 스프링은 왜 읽기 전용 트랜잭션을 강제하지 않았을까?

스프링의 핵심 설계 철학은 **유연함과 호환성**이다.<br/>
즉, 다양한 DBMS, JDBC 드라이버, 트랜잭션 매니저 환경에서도 동작할 수 있도록 `readOnly = true`는 **단지 힌트**로만 처리된다.<br/>
**강제 여부는 각 구현체가 결정**하며, 개발자가 필요에 따라 명시적으로 강제할 수 있도록 설계된 것이다.

이를 가능하게 해주는 설정이 바로 `enforceReadOnly`다.

```java
DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
transactionManager.setEnforceReadOnly(true);
```

`enforceReadOnly = true`로 설정하면, `DataSourceTransactionManager`는 내부적으로 `SET TRANSACTION READ ONLY` 쿼리를 실행하여 **DB 수준에서 실제로 읽기 전용 트랜잭션을 강제**한.

## JDBC 드라이버 동작 비교

스프링에서 전달한 `readOnly` 속성은 최종적으로 **JDBC 드라이버가 해석**한다.<br/>
JDBC는 `Connection.setReadOnly(boolean readOnly)` 메서드를 통해 **트랜잭션의 읽기 전용 여부를 설정**할 수 있는 표준 인터페이스를 제공한다.

**실제로 어떤 동작을 수행할지는 DBMS의 JDBC 드라이버 구현에 따라 달라진다.**

즉, 어떤 드라이버는 `readOnly` 설정을 무시하고, 어떤 드라이버는 이를 엄격하게 적용하여 쓰기 작업을 차단할 수도 있다.

### H2 Database

H2에서는 `JdbcConnection` 클래스가 `Connection` 인터페이스를 구현한다.<br/>
해당 클래스의 `setReadOnly()` 메서드는 다음과 같이 정의되어 있다.

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

보시는 것처럼, 이 메서드는 내부적으로 **아무런 동작도 하지 않는다.**<br/>
즉, `readOnly = true`로 설정하더라도 JDBC 수준에서 `SET TRANSACTION READ ONLY` 쿼리가 실행되지 않는다.

결과적으로 H2에서는 **읽기 전용 트랜잭션이 실제로 강제되지 않으며**, 트랜잭션 내에서 **쓰기 작업이 수행되더라도 예외가 발생하지 않는다.**

### MySQL

MySQL을 사용해서 테스트하면, `@Transactional(readOnly = true)` 설정 상태에서 **쓰기 작업이 실행될 경우 `TransientDataAccessResourceException` 예외가
발생**하고, 테스트가 성공한다.

MySQL에서는 `Connection` 인터페이스의 구현체로 `ConnectionImpl`을 사용하며, `setReadOnly()` 메서드는 다음과 같이 정의되어 있다.

```java
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

        this.readOnly = readOnlyFlag;
    }
}
```

`readOnly = true`로 설정되면, 내부적으로 `SET SESSION TRANSACTION READ ONLY` SQL이 실행되어 실제로 **DB 세션 수준에서 읽기 전용 모드가 적용**된다.

특히 MySQL은 스프링의 `enforceReadOnly` 설정과는 무관하게, 자체적으로 쓰기 작업을 차단하는 로직을 포함하고 있다.

예를 들어, 실제로 쿼리를 실행시키는 `ClientPreparedStatement` 클래스의 `executeUpdateInternal()` 메서드를 보면 다음과 같이 동작한다.

```java
protected long executeUpdateInternal(String sql, boolean isBatch, boolean returnGeneratedKeys) throws SQLException {
    synchronized (checkClosed().getConnectionMutex()) {
        // ...

        if (locallyScopedConn.isReadOnly(false)) {
            throw SQLError.createSQLException(Messages.getString("Statement.42") + Messages.getString("Statement.43"),
                    MysqlErrorNumbers.SQL_STATE_ILLEGAL_ARGUMENT, getExceptionInterceptor());
        }

        return this.updateCount;
    }
}
```

즉, **업데이트 쿼리 실행 시점에 연결이 읽기 전용인지 확인하고**, `readOnly = true` 상태일 경우 **JDBC 드라이버 자체에서 예외를 발생시킨다.**

따라서 MySQL을 사용하는 경우에는 `@Transactional(readOnly = true)` 설정만으로도 **쓰기 작업을 방지할 수 있다.**

## 정리

- `@Transactional(readOnly = true)`는 **트랜잭션이 읽기 전용임을 나타내는 힌트**일 뿐이며, **쓰기 작업을 강제적으로 차단하지 않는다.**
- **쓰기 작업이 실제로 차단되는지는 트랜잭션 매니저, JDBC 드라이버, DBMS의 구현 방식에 따라 다르게 동작한다.**
- 스프링은 설계 철학인 **유연함과 호환성**에 따라, `readOnly` 설정을 단순한 힌트로 처리하고, **강제하고 싶은 경우 `enforceReadOnly` 설정을 통해 명시적으로 적용할 수 있도록**지원한다.
- 최종적으로는 **JDBC 드라이버가 `readOnly` 속성을 어떻게 해석하느냐에 따라 동작이 결정**되므로, 사용 중인 DB와 드라이버의 구현 방식을 반드시 확인해야 한다.
