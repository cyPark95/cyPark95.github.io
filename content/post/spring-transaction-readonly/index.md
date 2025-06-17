---
title: Spring @Transactional(readOnly = true) 정말 읽기 전용일까?
description: 
date: 2025-06-18 00:00:00+0000
math: true
tags:
  - Spring
  - Transaction
  - readOnly
---

Spring에서 `@Transactional(readOnly = true)`를 사용하면 "읽기 전용" 트랜잭션이 실행된다고 알고 있지만 실제로 이 설정은 기대하는 것처럼 동작하지 않을 수 있다. 
그래서 `readOnly` 속성의 진짜 의미와 그 동작 방식, 그리고 DB 및 드라이버에 따라 어떤 차이가 있는지 알아 볼 것이다.

### `@Transactional(readOnly = true)`의 의미

Spring 공식 문서 중 [Transactional의 readOnly](https://docs.spring.io/spring-framework/docs/4.3.x/javadoc-api/org/springframework/transaction/annotation/Transactional.html#readOnly--)를 보면 다음과 같은 설명이 있다.
> This just serves as a hint for the actual transaction subsystem; 
> it will not necessarily cause failure of write access attempts. 
> A transaction manager which cannot interpret the read-only hint will not throw an exception when asked for a read-only transaction but rather silently ignore the hint.

즉, `readOnly = true`는 트랜잭션 서브 시스템에 대한 힌트일 뿐 반드시 쓰기 작업을 막아주는 것은 아니고

해당 힌트를 무시하는 트랜잭션 매니저나 JDBC 드라이버의 경우, 여전히 데이터 수정이 가능할 수 있다고 한다.

### 스프링의 트랜잭션 흐름

Spring은 `DataSourceTransactionManager`를 통해 JDBC 기반의 트랜잭션을 관리한다. 
트랜잭션이 시작되는 지점은 `doBegin()` 메서드로, 내부에서 `readOnly` 설정을 다음과 같이 처리한다.

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

따라서 쓰기 작업을 강제적으로 막고 싶다면 `enforceReadOnly` 속성을 함께 설정해야 한다.

### H2 Database

H2를 사용하는 환경에서 `@Transactional(readOnly = true)` 설정 후 update를 실행해보면, 예외가 발생하지 않고 데이터가 수정된다.
이는 H2 JDBC 드라이버의 구현체에서 이유를 찾을 수 있다.

```java
@Override
public void setReadOnly(boolean readOnly) throws SQLException {
    try {
        checkClosed();
        // 내부적으로 아무 동작도 하지 않음
    } catch (Exception e) {
        throw logAndConvert(e);
    }
}
```

`setReadOnly(true)`가 호출되더라도 실제로는 아무 일도 하지 않으며, 트랜잭션이 읽기 전용으로 전환되지 않는다. 
따라서 `enforceReadOnly`를 설정해도 `SET TRANSACTION READ ONLY`가 무시되고, 쓰기 작업이 가능한 것이다.

### MySQL

하지만 MySQL을 사용하면 `enforceReadOnly`를 `false`로 설정했음에도 update를 시도하면 `TransientDataAccessResourceException` 예외가 발생한다.
MySQL 드라이버 내부 구현을 보면 그 이유를 확인할 수 있다.

```java
if (locallyScopedConn.isReadOnly(false)) {
    throw SQLError.createSQLException("...", ...);
}
```

MySQL 드라이버는 쿼리 실행 시점에 연결 객체의 read-only 상태를 체크하고, 읽기 전용 상태일 경우 예외를 발생시킨다. 
이는 `setReadOnly(true)` 호출이 실제로 JDBC 커넥션 수준에서 강제되기 때문에 가능한 동작이다.

### 결론: `readOnly = true`의 진짜 의미

`@Transactional(readOnly = true)`는 트랜잭션 서브 시스템에 "읽기 전용일 가능성이 높다."는 힌트를 제공한다.
쓰기 작업을 강제로 막으려면 enforceReadOnly = true도 함께 설정해야 하며, 이것이 JDBC 드라이버에서 제대로 지원돼야 한다.
H2처럼 이를 무시하는 드라이버도 있으므로, 실제 효과는 DB와 드라이버에 따라 다르다.

### 마무리

개발 중 테스트나 로컬 환경에서 H2를 자주 사용하는데, 이 때문에 readOnly가 기대대로 동작하지 않아 혼란스러울 수 있다. 
이 설정이 정말로 보호막이 되려면, 드라이버의 지원 여부까지 확인하는 습관이 필요하다.
읽기 전용 트랜잭션을 올바르게 활용하기 위해서는 단순히 `@Transactional(readOnly = true)`만 붙이는 것으로는 부족하다. 
상황에 따라 `enforceReadOnly`와 DB 드라이버의 특성을 고려한 전략이 필요하다.
