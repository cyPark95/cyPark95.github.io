---
title: Spring @Transactional(readOnly = true) 정말 읽기 전용일까?
description: Spring 트랜잭션의 read-only 속성이 실제로 어떻게 동작하는지, JDBC 드라이버에 따라 어떻게 다르게 동작하는지 알아본다.
date: 2025-06-18 00:00:00+0000
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

다음 테스트 코드의 결과는 실패였고, 이유는 **예외가 발생하지 않았기 때문**이다.

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

@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    void readOnlyUpdate() {
        assertThrows(Exception.class, () -> userService.allUpgradeLevel());
    }
}
```

왜 예외가 발생하지 않았을까?

## 스프링의 `@Transactional` 어노테이션과 `readOnly` 속성

스프링의 `@Transactional` 어노테이션은 프록시 기반의 AOP(Aspect-Oriented Programming)를 통해 트랜잭션을 적용하는 기능이다.<br/>
메서드 실행 전/후에 트랜잭션을 시작하고, 정상적으로 끝나면 커밋(commit), 예외가 발생하면 롤백(rollback)하는 방식이다.

`@Transactional` 어노테이션은 트랜잭션 관련 다양한 속성을 제공한다.

- propagation: 트랜잭션 전파 방식
- isolation: 트랜잭션 격리 수준
- timeout: 트랜잭션 제한 시간
- rollbackFor / noRollbackFor: 롤백 예외 지정
- readOnly: 읽기 전용 트랜잭션 여부

그 중 `readOnly` 속성을 true로 설정하면 **"읽기 전용" 트랜잭션이 실행**된다고 알고 있지만, **실제로 그렇게 강제되지 않는다.**

[Spring 공식 문서](https://docs.spring.io/spring-framework/docs/4.3.x/javadoc-api/org/springframework/transaction/annotation/Transactional.html#readOnly--)
에서는 `readOnly` 속성을 다음과 같이 명시하고 있다.

> This just serves as a hint for the actual transaction subsystem;
> it will not necessarily cause failure of write access attempts.
> A transaction manager which cannot interpret the read-only hint will not throw an exception when asked for a read-only
> transaction but rather silently ignore the hint.

쉽게 말해, `readOnly = true`는 **힌트**일 뿐, 이를 강제하는 책임은 **트랜잭션 매니저와 JDBC 드라이버에 있다.**

## 스프링의 트랜잭션 내부 동작

스프링은 `DataSourceTransactionManager`를 통해 JDBC 기반 트랜잭션을 관리한다.<br/>
트랜잭션이 시작되는 지점은 `doBegin()` 메서드로, `readOnly` 속성은 다음 과정으로 처리된다.

- 커넥션 획득
- `DataSourceUtils.prepareConnectionForTransaction()` 메서드 내부에서 `Connection.setReadOnly(true)` 메서드를 통해 JDBC 드라이버에 힌트 제공
- `prepareTransactionalConnection()` 메서드를 통해 읽기 전용 트랜잭션 설정

```java
protected void prepareTransactionalConnection(Connection con, TransactionDefinition definition) throws SQLException {
    if (isEnforceReadOnly() && definition.isReadOnly()) {
        try (Statement stmt = con.createStatement()) {
            stmt.executeUpdate("SET TRANSACTION READ ONLY");
        }
    }
}
```

특이한 점은 `readOnly` 설정 외에도 `enforceReadOnly` 속성을 확인한다는 점이다.<br/>

- `readOnly = true`만으로는 실제 DB 수준에서 읽기 전용 트랜잭션을 강제하지 않는다.
- `enforceReadOnly = true`를 설정해야만 `SET TRANSACTION READ ONLY` 쿼리가 실행되어 **DB 수준에서 읽기 전용 트랜잭션 강제한다.**

## 스프링은 왜 강제하지 않았을까?

스프링의 기본 설계 철학은 **유연함과 호환성**을 제공하는 데 있다.<br/>
다양한 DB, 다양한 드라이버, 다양한 트랜잭션 매니저에 자율성을 부여할 수 있도록 `readOnly = true`는 단순한 힌트로 처리하고, 강제하고 싶은 경우 개발자가 명시적으로 설정하도록 설계되어 있다.

그래서 등장한 것이 `enforceReadOnly` 속성이다.

```java
DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
transactionManager.setEnforceReadOnly(true);
```

`enforceReadOnly = true`로 설정해야 `DataSourceTransactionManager`는 내부적으로 쿼리를 실행하여 다음과 같이 실제 DB 수준에서 읽기 전용 트랜잭션을 강제하게
된다.

## 실제 JDBC 드라이버 동작 비교

스프링에서 전달한 `readOnly` 속성은 결국 JDBC 드라이버가 해석한다.<br/>
JDBC는 `Connection.setReadOnly(boolean readOnly)` 메서드를 통해 트랜잭션의 읽기 전용 여부를 설정할 수 있도록 인터페이스를 제공한다.

이 메서드는 표준 인터페이스로 정의되어 있지만, **실제 동작은 각 DBMS의 JDBC 드라이버 구현에 따라 달라진다.**

### H2 Database

H2에서는 `JdbcConnection`을 통해 `Connection` 인터페이스를 구현한다.<br/>
`JdbcConnection.setReadOnly()` 메서드는 다음과 같다.

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

내부적으로 **아무런 동작도 하지 않는다.**<br/>
따라서 `readOnly = true`로 설정해도 `SET TRANSACTION READ ONLY` 쿼리가 실행되지 않는다.<br/>
그 결과 **쓰기 작업이 가능**하며 예외가 발생하지 않는다.

### MySQL

MySQL을 사용해서 테스트 해보면 `TransientDataAccessResourceException` **예외가 발생하고, 테스트가 성공한다.**

MySQL은 `ConnectionImpl`을 통해 `Connection` 인터페이스를 구현한다.<br/>
`ConnectionImpl.setReadOnly()` 메서드는 다음과 같다.

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
        // ...

        this.readOnly = readOnlyFlag;
    }
}
```

`readOnly = true`로 설정하면 내부적으로 `SET SESSION TRANSACTION READ ONLY` 쿼리가 실행되어 **읽기 전용 모드를 설정한다.**

또한 `enforceReadOnly` 속성과 상관 없이 **업데이트 쿼리 실행 시점에 예외가 발생한다.**

MySQL은 `ClientPreparedStatement` 클래스를 통해 실제로 쿼리를 실행한다.<br/>
실제 쿼리를 실행시키는 `executeUpdateInternal()` 메서드는 다음 같다.

```java
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
```

MySQL 드라이버는 업데이트 쿼리 실행 시점에 연결 객체의 `readOnly = true` 상태를 체크하고, **읽기 전용 상태일 경우 자체적으로 예외를 발생**시킨다.<br/>
따라서 MySQL을 사용하면 `readOnly = true` 설정만으로도 **쓰기 작업 시점에 예외가 발생한다.**
