---
title: Spring @Transactional(readOnly = true)는 정말 읽기 전용 트랜잭션일까?
description: JDBC 드라이버의 구현에 따라 readOnly의 동작이 달라지는 이유를 알아본다.
date: 2025-06-19 00:00:00+0000
categories:
  - Spring
tags:
  - Transaction
  - JDBC
  - JPA/Hibernate
  - readOnly
---

스프링에서는 `@Transactional(readOnly = true)`를 통해 읽기 전용 트랜잭션을 설정할 수 있다.<br/>
실제로 이 설정이 "쓰기 작업"을 막아줄까?

다음은 `@Transactional(readOnly = true)` 설정에서 JDBC를 사용하여 사용자의 레벨을 수정하는 간단한 코드다.

```java
@Transactional(readOnly = true)
public void upgradeLevel(long userId) throws SQLException {
    try (
            Connection conn = dataSource.getConnection();
            PreparedStatement ps = conn
                .prepareStatement("UPDATE users SET level = level + 1 WHERE id = ?")
    ) {
        ps.setLong(1, userId);
        ps.executeUpdate();
    }
}
```

쓰기 작업이 발생하면 예외가 발생할 것이라 기대하고 테스트를 작성했다.

```java
@DisplayName("readOnly = true인 트랜잭션에서 쓰기 작업이 발생하면 예외가 발생해야 한다.")
@Test
void readOnlyUpdate() {
    assertThrows(Exception.class, () -> userService.upgradeLevel(1L));
}
```

하지만 H2를 사용한 테스트에서는 예외가 발생하지 않아 테스트가 실패했다.

이번 포스팅에서는 왜 테스트에 실패했는지와 `readOnly` 속성이 어떻게 처리되는지 알아볼 것이다.

## readOnly = true는 무엇을 보장할까?

이 질문에 답하기 위해서는 먼저 Spring이 `readOnly` 속성을 어떻게 처리하는지 알아야 한다.

[Spring 공식 Java 문서](https://docs.spring.io/spring-framework/docs/4.3.x/javadoc-api/org/springframework/transaction/annotation/Transactional.html#readOnly--)에서는 다음과 같이 명시하고 있다.

> This just serves as a hint for the actual transaction subsystem; it will not necessarily cause failure of write access attempts.

즉, `readOnly = true`는 단지 **트랜잭션 시스템에 전달되는 힌트**일 뿐이며, 실제로 쓰기 작업을 막을지 여부는 **트랜잭션 매니저와 JDBC 드라이버의 구현에 달려 있다.**

## Spring에서 readOnly 처리 방식

Spring의 TransactionManager가 `readOnly` 속성을 어떻게 처리하는지 살펴보자.

```java
// Spring의 TransactionManager 내부 동작
public void doBegin(TransactionDefinition definition) {
    Connection conn = dataSource.getConnection();
    
    if (definition.isReadOnly()) {
        conn.setReadOnly(true);
    }
    
    conn.setAutoCommit(false);
}
```

Spring은 `readOnly = true`를 받으면 JDBC 레벨의 `Connection.setReadOnly(true)`를 호출한다. 

**여기서 중요한 것은 Spring이 직접 처리하지 않는다는 것이다.** Spring은 단지 JDBC 표준 인터페이스를 호출하고, 실제 동작은 JDBC 드라이버에 위임하는 것이다.

```java
Connection conn = dataSource.getConnection();
conn.setReadOnly(true);
```

표준 JDBC 인터페이스에서 제공하는 `Connection.setReadOnly(boolean readOnly)` 메서드다. 하지만 **이것이 실제로 어떻게 동작할지는 JDBC 드라이버의 구현에 따라 달라진다.**

### DB 레벨 읽기 전용 강제 enforceReadOnly

하지만 JDBC 드라이버에 따라 readOnly의 효과가 달라질 수 있으므로, Spring은 DataSourceTransactionManager에서 `enforceReadOnly` 옵션을 제공한다.

```java
DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
transactionManager.setEnforceReadOnly(true);
```

`enforceReadOnly = true`로 설정하면 Spring은 내부적으로 다음과 같이 SQL을 직접 실행하여 DB 레벨에서 실제로 읽기 전용을 강제한다.

```java
// Spring이 enforceReadOnly = true일 때 내부적으로 실행하는 코드
if (isEnforceReadOnly() && definition.isReadOnly()) {
    try (Statement stmt = conn.createStatement()) {
        stmt.executeUpdate("SET TRANSACTION READ ONLY");
    }
}
```

readOnly의 효과가 JDBC 드라이버에 따라 달라질 수 있으므로, Spring이 이 옵션을 제공한 것은 진정한 읽기 전용을 원하는 개발자들을 위한 선택지를 주려는 의도가 아닐까?

다만 Spring Data JPA 환경에서는 JpaTransactionManager를 사용하므로 enforceReadOnly 옵션을 직접 사용할 수 없다.

## JDBC 드라이버 동작 비교

이제 왜 테스트 결과가 다를까를 이해할 수 있다. JDBC 드라이버의 `setReadOnly()` 구현을 비교해보자.

### H2 - 왜 테스트가 실패했을까?

H2의 `JdbcConnection` 클래스에서 `setReadOnly()` 메서드는 다음과 같이 구현되어 있다.

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

보시는 것처럼, 이 메서드는 단순히 연결이 닫혀있는지만 확인할 뿐 **readOnly 속성을 실제로 처리하지 않는다.**<br/>
즉, `setReadOnly(true)`를 호출해도 H2는 실제로 읽기 전용 상태를 강제하지 않는 것이다.

결과적으로 H2에서는:
- `readOnly = true`로 설정되어도 JDBC 수준에서 읽기 전용이 강제되지 않음
- 트랜잭션 내에서 **쓰기 작업이 수행되어도 예외가 발생하지 않음**
- 따라서 테스트가 실패함

### MySQL - 왜 테스트가 성공할까?

MySQL의 `ConnectionImpl` 클래스에서 `setReadOnly()` 메서드는 다음과 같이 구현되어 있다.

```java
@Override
public void setReadOnly(boolean readOnlyFlag) throws SQLException {
    setReadOnlyInternal(readOnlyFlag);
}

@Override
public void setReadOnlyInternal(boolean readOnlyFlag) throws SQLException {
    synchronized (getConnectionMutex()) {
        this.session.execSQL(null, "SET SESSION TRANSACTION " + (readOnlyFlag ? "READ ONLY" : "READ WRITE"), -1, null, false,
                this.nullStatementResultSetFactory, null, false);

        this.readOnly = readOnlyFlag;
    }
}
```

MySQL은 `setReadOnly(true)`를 호출하면 **실제로 `SET SESSION TRANSACTION READ ONLY` SQL을 실행한다.**<br/>
이 SQL이 MySQL 서버 레벨에서 읽기 전용 상태를 강제한다.

특히 MySQL의 `ClientPreparedStatement` 클래스의 `executeUpdateInternal()` 메서드를 보면 다음과 같이 동작한다.

```java
protected long executeUpdateInternal(String sql, boolean isBatch, boolean returnGeneratedKeys) throws SQLException {
    synchronized (checkClosed().getConnectionMutex()) {
        if (locallyScopedConn.isReadOnly(false)) {
            throw SQLError.createSQLException(Messages.getString("Statement.42") + Messages.getString("Statement.43"),
                    MysqlErrorNumbers.SQL_STATE_ILLEGAL_ARGUMENT, getExceptionInterceptor());
        }

        return this.updateCount;
    }
}
```

즉, **쿼리 실행 시점에 연결이 읽기 전용인지 확인하고**, 읽기 전용 상태일 경우 **JDBC 드라이버 자체에서 예외를 발생시킨다.**

결과적으로 MySQL에서는:
- `readOnly = true`로 설정하면 `SET SESSION TRANSACTION READ ONLY` SQL이 실행됨
- 실제로 MySQL 서버 레벨에서 읽기 전용이 강제됨
- 트랜잭션 내에서 **쓰기 작업 시도 시 예외가 발생**
- 따라서 테스트가 성공함

### JDBC 드라이버 구현에 따른 차이

| 드라이버 | setReadOnly() 구현 | 결과 |
|---------|------------------|------|
| **H2** | 연결 상태만 확인 (readOnly 속성 처리 없음) | 쓰기 SQL 실행됨, 예외 없음 |
| **MySQL** | `SET SESSION TRANSACTION READ ONLY` SQL 실행 | 쓰기 SQL 차단됨, 예외 발생 |

같은 JDBC 메서드를 호출하지만, 드라이버 구현에 따라 완전히 다른 결과가 나타난다.

## JPA/Hibernate에서의 readOnly 동작

이제 JDBC 드라이버의 구현 차이를 알아봤으니, 그렇다면 JPA/Hibernate에서는 어떻게 처리하고 있는지 살펴보자.

JpaTransactionManager는 트랜잭션을 시작할 때 Hibernate의 `HibernateJpaDialect`에 처리를 위임한다. `readOnly = true`라면 다음과 같은 일이 일어난다.

```java
protected @Nullable FlushMode prepareFlushMode(
        Session session, boolean readOnly) {

    FlushMode flushMode = session.getHibernateFlushMode();

    if (readOnly) {
        if (!flushMode.equals(FlushMode.MANUAL)) {
            session.setHibernateFlushMode(FlushMode.MANUAL);
            return flushMode;
        }
    }

    return null;
}
```

`readOnly = true`일 때 Hibernate는 두 가지를 한다:
1. **`FlushMode.MANUAL` 설정** → 자동 flush 억제
2. **`Connection.setReadOnly(true)` 호출** → 위에서 본 JDBC 드라이버 메서드 호출

### 자동 flush의 억제

일반적인 트랜잭션에서는 커밋 시점에 자동 flush가 발생한다.

```text
엔티티 변경
    ↓
트랜잭션 커밋
    ↓
자동 flush (Hibernate가 변경 감지)
    ↓
UPDATE SQL 실행
```

하지만 readOnly 트랜잭션에서는 자동 flush가 억제된다.

```text
엔티티 변경
    ↓
트랜잭션 커밋
    ↓
자동 flush 생략 (FlushMode.MANUAL)
    ↓
UPDATE SQL이 실행되지 않음
```

그 결과 일반적인 `save()` 호출이나 엔티티 변경만으로는 UPDATE SQL이 실행되지 않는다.

### 명시적 flush는?

Hibernate가 자동 flush를 억제해도, 명시적으로 flush를 호출하면 UPDATE SQL이 실행을 시도한다.

```java
@Transactional(readOnly = true)
public void updateUser() {
    User user = userRepository.findById(1L).orElseThrow();
    user.addLevel();
    
    entityManager.flush();
}
```

또는 `saveAndFlush()`를 사용할 수도 있다.

```java
userRepository.saveAndFlush(user);
```

**이때부터는 JDBC 드라이버의 `setReadOnly()` 구현이 작동한다.**
- **H2**: 구현이 비어있으므로 UPDATE SQL이 실행됨
- **MySQL**: `SET SESSION TRANSACTION READ ONLY`가 유효하므로 예외 발생

## 정리

- `readOnly = true`는 트랜잭션 시스템에 전달되는 단순한 힌트일 뿐이다.
- Spring은 이 힌트를 JDBC 드라이버의 `Connection.setReadOnly(true)`로 위임한다.
- JDBC 드라이버 구현에 따라 readOnly의 효과가 달라진다. H2는 무시하고, MySQL은 실제로 읽기 전용을 강제한다.
- Spring Data JPA + Hibernate 환경에서는 `FlushMode.MANUAL`을 설정하여 자동 flush를 억제한다.
- 명시적으로 flush한 쓰기 SQL을 차단할지는 JDBC 드라이버와 DBMS 구현에 달려 있다.

결국 `readOnly = true`는 쓰기 방지 장치라기보다, **조회 전용 트랜잭션이라는 의도를 표현하고 불필요한 flush를 줄이는 최적화 설정**에 가깝다.

## 레퍼런스

- [Spring Javadoc: @Transactional](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html)
- [Hibernate FlushMode 문서](https://docs.hibernate.org/orm/7.0/javadocs/org/hibernate/FlushMode.html)
- [MySQL Connector/J Source Code](https://github.com/mysql/mysql-connector-j)
