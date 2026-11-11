---
title: "커버링 인덱스(Covering Index)의 동작 원리와 InnoDB PK 자동 포함 구조, 그리고 JPA 실무 적용법"
date: 2026-11-11T19:50:00+09:00
draft: false
summary: "WHERE 절을 넘어 쿼리 전체(SELECT, ORDER BY, GROUP BY)를 아우르는 커버링 인덱스의 동작 원리와 디스크 랜덤 I/O 제거 메커니즘, InnoDB 세컨더리 인덱스의 PK 자동 포함 특성, 그리고 Spring Data JPA에서 커버링 인덱스를 적용하는 프로젝션 방법을 정리합니다."
categories: ["Backend"]
tags: ["Database", "MySQL", "InnoDB", "CoveringIndex", "B+Tree", "JPA", "Querydsl", "QueryTuning"]
---

## 1. 인덱스 설계는 `WHERE` 절만이 아니라 '쿼리 전체'를 봐야 한다

일반적으로 인덱스를 설계한다고 하면 `WHERE` 절에 들어가는 조건 컬럼만 떠올리기 쉽지만, 실제로는 <strong>`WHERE`뿐만 아니라 `SELECT`, `ORDER BY`, `GROUP BY`까지 쿼리 전체에 대한 인덱스 설계</strong>가 필요하다.

인덱스는 데이터를 효율적으로 찾는 수단이지만, MySQL(InnoDB)의 경우 인덱스 리프 노드 안에 포함된 컬럼 데이터를 그대로 꺼내 쓸 수 있다. 따라서 이를 잘 활용하면 **실제 테이블 데이터 블록까지 접근할 필요가 전혀 없어진다.**

---

## 2. 커버링 인덱스(Covering Index)란 무엇인가?

- 쿼리를 충족시키는 데 필요한 모든 데이터를 갖고 있는 인덱스를 <strong>커버링 인덱스(Covering Index 혹은 Covered Index)</strong>라고 부른다.
- 즉, 쿼리 실행에 필요한 모든 컬럼(`SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`)이 **인덱스 B+Tree 자체에 100% 포함되어 있는 인덱스**를 뜻한다.

### InnoDB 세컨더리 인덱스와 기본키(PK)의 관계

여기서 중요한 점은 InnoDB 스토리지 엔진이 세컨더리 인덱스(보조 인덱스)와 PK를 다루는 물리적 구조다.

- **원리**: MySQL InnoDB는 물리적으로 모든 세컨더리 인덱스의 리프 노드 맨 끝에 **테이블의 기본키(PK, Clustered Key)를 자동으로 붙여서 저장**한다.
- 예를 들어 인덱스를 `(email, first_name)`으로만 생성했더라도, 실제 물리적 저장 형태는 <strong>`(email, first_name, id)`</strong>가 된다.
- 따라서 인덱스에 `id`를 명시하지 않았더라도 `SELECT id, email, first_name ...`이나 `SELECT id FROM payments WHERE user_id = ?`처럼 PK가 포함된 조회를 할 때 **데이터 블록에 가지 않고 100% 커버링 인덱스(`Using index`)로 동작**한다.

```sql
-- (email, first_name) 인덱스만 있어도 리프 노드 끝에 PK(id)가 자동 포함되어 있음
SELECT id, email, first_name
FROM users
WHERE email = 'user@example.com';

-- (user_id) 인덱스만 있어도 PK(id) 조회는 커버링 인덱스로 처리됨
SELECT id
FROM payments
WHERE user_id = 12345;
```

---

## 3. 일반 인덱스 조회 vs 커버링 인덱스 조회의 디스크 동작 비교

MySQL InnoDB의 데이터 저장 구조는 크게 2개의 B+Tree로 나뉜다.

1. **세컨더리 인덱스 (보조 인덱스)**: 우리가 `CREATE INDEX`로 만드는 일반적인 인덱스로, <strong>[인덱스 키 컬럼 + PK]</strong>가 저장된다.
2. **클러스터드 인덱스 (Clustered Index)**: PK 순서대로 정렬되어 있으며, **실제 데이터 행(Row) 전체**가 저장된 본체다.

`payments` 테이블에 인덱스가 걸려 있을 때 일반 조회와 커버링 인덱스 조회가 디스크 I/O 측면에서 어떻게 다른지 살펴보자.

### 1) 일반 인덱스 조회 (`SELECT *`)

인덱스가 `(user_id, paid_at)`으로 만들어져 있을 때 다음 쿼리를 실행한다고 가정해 보자.

```sql
SELECT * FROM payments WHERE user_id = 12345;
```

이때 DB 엔진은 두 개의 B+Tree를 **2번 왕복**해야 한다.

```text
[1단계: 세컨더리 인덱스 B+Tree]
루트 ➔ 브랜치 ➔ 리프 노드 도착
여기서 user_id = 12345 인 데이터 1만 건의 "결제 PK(id)" 목록을 뽑아냄
   │
   ▼ (문제: 뽑아낸 PK 1만 개를 들고 진짜 데이터를 찾으러 출발)
[2단계: 클러스터드 인덱스 B+Tree (실제 데이터 파일)]
PK 1번 행 찾으러 디스크 접근 (랜덤 I/O)
PK 45번 행 찾으러 디스크 접근 (랜덤 I/O)
PK 8900번 행 찾으러 디스크 접근 (랜덤 I/O)
... ➔ 1만 번의 디스크 무작위 접근(랜덤 I/O) 발생
```

- **왜 느릴까?**: `SELECT *` 때문에 인덱스에 없는 나머지 컬럼들을 다 읽어야 한다. 그런데 이 실제 데이터 행들은 디스크 블록 여기저기에 흩어져 있어서, 결국 **랜덤 I/O를 1만 번 수행**해야 하기 때문이다.

### 2) 커버링 인덱스 조회

이번에는 인덱스가 `(user_id, paid_at, amount)`로 만들어져 있고, 쿼리가 요구하는 모든 컬럼이 인덱스에 포함된 경우다.

```sql
SELECT user_id, paid_at, amount
FROM payments
WHERE user_id = 12345;
```

동작 과정은 다음과 같다.

1. **옵티마이저 쿼리 분석**: `SELECT`에서 요구하는 컬럼이 `user_id`, `paid_at`, `amount` 3개임을 확인한다.
2. **인덱스 정의 확인**: `(user_id, paid_at, amount)` 인덱스 리프 노드에 이미 해당 3개 데이터가 모두 들어있는 것을 확인한다.
3. **즉시 반환**: 실제 테이블(클러스터드 인덱스)로 갈 필요 없이 리프 노드에 있는 값을 그대로 읽어서 바로 반환한다.

이 과정을 통해 수만 번 발생하던 <strong>디스크 랜덤 I/O 비용이 '0'</strong>이 된다.

---

## 4. 실행 계획(`EXPLAIN`) 판별법

쿼리가 커버링 인덱스로 동작하는지는 `EXPLAIN` 실행 계획의 `Extra` 항목으로 판별할 수 있다.

- **`Using index`**: 커버링 인덱스가 적용된 상태다. 실제 테이블 데이터 블록에 접근하지 않고 인덱스 B+Tree 리프 노드의 데이터만 읽어서 처리를 끝냈다는 의미다.
- **`Using index condition` (참고)**: 문구가 비슷하지만 커버링 인덱스가 아니다. 인덱스에서 조건을 먼저 필터링한 뒤, 나머지 데이터를 읽기 위해 결국 테이블로 접근한다는 ICP(Index Condition Pushdown)를 의미한다.

---

## 5. Spring Data JPA에서 주의할 점과 적용 방법

### 1) 안티패턴: 엔티티 자체를 조회하는 경우

```java
// JPA 기본 메서드: SELECT * FROM payments WHERE user_id = ?
List<Payment> payments = paymentRepository.findByUserId(12345L);
```

JPA로 엔티티 자체(`Payment`)를 조회하면 무조건 <strong>`SELECT *`</strong>로 쿼리가 나가기 때문에, DB에 복합 인덱스를 아무리 잘 만들어두어도 커버링 인덱스의 이점을 전혀 살리지 못한다.

### 2) 최적화: 프로젝션(Projection)을 통한 필요한 컬럼만 조회

커버링 인덱스를 적용하기 위해 굳이 위험 부담이 있는 Native Query를 쓸 필요는 없다. 아래 방식들처럼 필요한 컬럼만 지정해서 조회(프로젝션)하면 JPA 환경에서도 그대로 커버링 인덱스를 태울 수 있다.

#### ① JPQL `new` 명령어를 활용한 DTO 프로젝션

```java
// 필요한 컬럼만 DTO로 지정하여 조회 -> 커버링 인덱스 동작
@Query("SELECT new com.dto.PaymentDto(p.userId, p.paidAt, p.amount) " +
       "FROM Payment p WHERE p.userId = :userId")
List<PaymentDto> findSummaryByUserId(@Param("userId") Long userId);
```

#### ② 인터페이스 기반 프로젝션 (`getter`만 정의한 인터페이스 반환)

```java
public interface PaymentSummary {
    Long getUserId();
    LocalDateTime getPaidAt();
    BigDecimal getAmount();
}

// 반환 타입을 인터페이스로 지정하면 정의된 getter 컬럼만 SELECT 절에 나감
List<PaymentSummary> findByUserId(Long userId);
```

#### ③ Querydsl DTO 프로젝션 (`Projections.constructor`)

```java
List<PaymentDto> result = queryFactory
    .select(Projections.constructor(PaymentDto.class,
        payment.userId,
        payment.paidAt,
        payment.amount
    ))
    .from(payment)
    .where(payment.userId.eq(userId))
    .fetch();
```

---

## 6. 요약 정리

1. **커버링 인덱스의 개념**: 쿼리 실행에 필요한 모든 컬럼(`SELECT`, `WHERE`, `ORDER BY` 등)이 인덱스 B+Tree 자체에 100% 포함되어 있어 실제 테이블 블록 접근(랜덤 I/O)을 생략하는 방식이다.
2. **InnoDB의 PK 자동 포함**: InnoDB 세컨더리 인덱스는 물리적으로 리프 노드 끝에 테이블의 PK를 자동으로 포함하여 저장한다. 따라서 `(user_id)` 인덱스만 있어도 `SELECT id FROM payments WHERE user_id = ?`는 인덱스에 `id`를 따로 명시하지 않아도 커버링 인덱스로 동작한다.
3. **실행 계획과 JPA 적용**: `EXPLAIN`의 `Extra` 항목에 <strong>`Using index`</strong>가 표시되는지 확인하고, JPA에서는 기본 엔티티 조회가 `SELECT *`로 나가므로 **인터페이스 프로젝션, JPQL `new` 구문, Querydsl DTO 프로젝션**을 사용해 필요한 컬럼만 조회해야 한다.
