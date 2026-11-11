---
title: "커버링 인덱스(Covering Index)의 동작 원리와 InnoDB PK 자동 포함 구조, 그리고 JPA 실무 적용법"
date: 2026-11-11T19:50:00+09:00
draft: false
summary: "WHERE 절을 넘어 쿼리 전체(SELECT, ORDER BY, GROUP BY)를 아우르는 커버링 인덱스의 동작 원리와 디스크 랜덤 I/O 제거 메커니즘, InnoDB 세컨더리 인덱스의 PK 자동 포함 특성, 그리고 Spring Data JPA에서 SELECT * 안티패턴을 피해 커버링 인덱스를 적용하는 프로젝션 전략을 정리합니다."
categories: ["Backend"]
tags: ["Database", "MySQL", "InnoDB", "CoveringIndex", "B+Tree", "JPA", "Querydsl", "QueryTuning"]
---

## 1. 인덱스 설계는 `WHERE` 절만이 아니라 '쿼리 전체'를 봐야 한다

일반적으로 인덱스를 설계한다고 하면 `WHERE` 절에 들어가는 검색 조건 컬럼을 어떻게 구성할지를 먼저 떠올린다. 하지만 실무에서 대용량 트래픽 쿼리를 극한으로 튜닝할 때는 <strong>`WHERE` 절뿐만 아니라 `SELECT`, `ORDER BY`, `GROUP BY`까지 쿼리 전체에 대한 인덱스 설계</strong>가 필요하다.

인덱스는 원하는 데이터의 위치를 빠르게 찾기 위한 구조이지만, MySQL(InnoDB)에서는 인덱스 리프 노드 안에 실제 컬럼 값들이 함께 저장되어 있다. 따라서 이 특성을 잘 활용하면 **실제 테이블 데이터 블록(디스크)에 아예 접근하지 않고 인덱스만 읽어서 쿼리를 끝내는 것**이 가능하다.

---

## 2. 커버링 인덱스(Covering Index)란 무엇인가?

- **정의**: 쿼리를 충족시키는 데 필요한 모든 데이터를 갖고 있는 인덱스를 <strong>커버링 인덱스(Covering Index 혹은 Covered Index)</strong>라고 부른다.
- 즉, 쿼리 실행에 사용되는 모든 컬럼(`SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`)이 **인덱스 B+Tree 자체에 100% 포함되어 있는 상태**를 의미한다.

### 핵심 원리: InnoDB 세컨더리 인덱스와 기본키(PK)의 물리적 결합

MySQL InnoDB 스토리지 엔진에서 커버링 인덱스를 설계할 때 반드시 알아야 할 구조적 특징이 있다.

> **InnoDB의 모든 세컨더리 인덱스(보조 인덱스)는 물리적으로 리프 노드 맨 끝에 테이블의 기본키(PK, Clustered Key) 값을 자동으로 붙여서 저장한다.**

세컨더리 인덱스는 실제 데이터 행(Row)이 있는 위치를 찾아가야 하는데, InnoDB에서는 그 주소값 역할로 물리적 디스크 주소 대신 **논리적 주소인 PK 값**을 들고 있기 때문이다.

- 예를 들어 개발자가 인덱스를 `(email, first_name)` 두 컬럼으로만 생성했더라도, 실제 InnoDB의 인덱스 리프 노드에는 <strong>`(email, first_name, id)`</strong> 형태로 PK(`id`)가 함께 저장된다.
- 마찬가지로 `(user_id)` 단일 인덱스만 걸려 있더라도 실제 리프 노드 구조는 <strong>`(user_id, id)`</strong>다.
- 따라서 인덱스에 `id` 컬럼을 명시적으로 포함시키지 않았더라도 아래와 같이 PK를 조회하는 쿼리는 **테이블 데이터 블록에 접근하지 않고 100% 커버링 인덱스(`Using index`)로 동작**한다.

```sql
-- (email, first_name) 인덱스만 존재해도 id는 리프 노드에 기본 포함됨
SELECT id, email, first_name
FROM users
WHERE email = 'user@example.com';

-- (user_id) 인덱스만 존재해도 PK(id) 조회는 100% 커버링 인덱스로 처리됨
SELECT id
FROM payments
WHERE user_id = 12345;
```

---

## 3. 일반 인덱스 조회 vs 커버링 인덱스 조회의 디스크 동작 비교

왜 인덱스를 똑같이 타는데도 일반 인덱스 조회와 커버링 인덱스 조회 사이에 수십 배의 성능 차이가 발생할까? 이를 이해하려면 MySQL InnoDB가 관리하는 **두 개의 B+Tree 구조**를 구분해야 한다.

1. **세컨더리 인덱스 (보조 인덱스)**: 우리가 `CREATE INDEX`로 생성하는 일반적인 인덱스로, 리프 노드에 <strong>[인덱스 키 컬럼 + PK]</strong>만 담겨 있다.
2. **클러스터드 인덱스 (Clustered Index)**: 테이블당 단 하나만 존재하며 PK 순서대로 정렬되어 있고, 리프 노드에 **실제 데이터 행(Row)의 모든 컬럼 데이터**가 저장된 테이블 본체다.

`payments` 테이블에 `(user_id, paid_at)` 인덱스가 걸려 있는 상황에서 두 쿼리의 디스크 I/O 과정을 비교해 보자.

### 1) 일반 인덱스 조회 (`SELECT *`) — 왜 느릴까?

```sql
-- 인덱스: (user_id, paid_at)
SELECT * FROM payments WHERE user_id = 12345;
```

이 쿼리를 실행하면 DB 엔진은 두 개의 B+Tree를 **2번 왕복**해야 한다.

```text
[1단계: 세컨더리 인덱스 B+Tree 탐색]
루트 ➔ 브랜치 ➔ 리프 노드 도착
여기서 user_id = 12345 조건에 맞는 데이터 1만 건의 "결제 PK(id)" 목록을 추출함
   │
   ▼ (문제: SELECT * 때문에 인덱스에 없는 나머지 컬럼들을 읽으러 실제 테이블로 가야 함)
[2단계: 클러스터드 인덱스 B+Tree (실제 데이터 파일 접근)]
PK 1번 행 찾으러 디스크 블록 접근 (랜덤 I/O)
PK 45번 행 찾으러 디스크 블록 접근 (랜덤 I/O)
PK 8900번 행 찾으러 디스크 블록 접근 (랜덤 I/O)
... ➔ 총 1만 번의 디스크 무작위 접근(Random I/O) 발생!
```

- **병목의 원인**: 세컨더리 인덱스에서는 `user_id = 12345`인 데이터들이 나란히 모여 있어 순차 스캔이 빠르지만, 거기서 뽑아낸 **PK 값(`1`, `45`, `8900`...)들은 클러스터드 인덱스 디스크 블록 여기저기에 무작위로 흩어져 있다.**
- 결국 `SELECT *`로 나머지 컬럼을 다 퍼 올리기 위해 디스크의 서로 다른 페이지를 1만 번 찌르는 <strong>랜덤 I/O(Random I/O)</strong>가 발생하면서 쿼리 속도가 급격히 느려진다.

### 2) 커버링 인덱스 조회 — 디스크 랜덤 I/O 비용 '0'

이번에는 쿼리가 실제로 화면에 보여줄 3개 컬럼(`user_id`, `paid_at`, `amount`)만 요청하고, 인덱스도 `(user_id, paid_at, amount)`로 구성되어 있다고 가정해 보자.

```sql
-- 인덱스: (user_id, paid_at, amount)
SELECT user_id, paid_at, amount
FROM payments
WHERE user_id = 12345;
```

이때 옵티마이저와 스토리지 엔진은 다음과 같이 동작한다.

```text
1. 쿼리 분석: 옵티마이저가 SELECT와 WHERE 절을 확인하니 필요한 컬럼은 user_id, paid_at, amount 3개뿐임
2. 인덱스 확인: (user_id, paid_at, amount) 세컨더리 인덱스 리프 노드에 이미 3개 컬럼 값이 모두 들어있음을 확인
3. 즉시 반환: 클러스터드 인덱스(실제 테이블 데이터 파일)로 갈 필요 없이,
              세컨더리 인덱스 리프 노드에 연속으로 모여 있는 값만 쓱 읽어서(Sequential Read) 바로 반환!
```

결과적으로 가장 비싼 작업이었던 **2단계(수만 번의 클러스터드 인덱스 랜덤 I/O) 비용이 완전히 '0'이 된다.**

---

## 4. 실행 계획(`EXPLAIN`)을 통한 커버링 인덱스 판별법

내가 작성한 쿼리가 커버링 인덱스로 동작하는지는 `EXPLAIN` 실행 계획의 **`Extra`** 컬럼으로 즉시 확인할 수 있다.

| `EXPLAIN`의 `Extra` 표시 | 의미 및 디스크 동작 |
| :--- | :--- |
| **`Using index`** | **커버링 인덱스 적용 성공!** 테이블 데이터 블록에 전혀 접근하지 않고 인덱스 B+Tree 리프 노드만 읽어서 쿼리를 완료했음을 뜻하는 최고의 신호다. |
| **`Using index condition`** | 이름이 비슷해 헷갈리기 쉽지만 커버링 인덱스가 아니다. <strong>ICP(Index Condition Pushdown)</strong>가 작동했다는 뜻으로, 인덱스에서 조건을 1차 필터링한 뒤 **결국 실제 테이블 데이터 블록을 읽으러 간다.** |

---

## 5. Spring Data JPA 환경에서의 실무 주의점과 최적화

DB에 커버링 인덱스를 아무리 잘 설계해 두었더라도, 백엔드 애플리케이션에서 **JPA를 기본 방식대로 사용하면 커버링 인덱스가 전부 깨지는 함정**에 빠지기 쉽다.

### 1) 안티패턴: 엔티티(`Entity`) 자체를 그대로 조회하는 경우

```java
// JPA 기본 쿼리 메서드 호출
List<Payment> payments = paymentRepository.findByUserId(12345L);
```

```sql
-- 실제 하이버네이트가 실행하는 SQL
SELECT p.id, p.user_id, p.paid_at, p.amount, p.pg_provider, p.card_number, p.created_at ...
FROM payments p
WHERE p.user_id = ?
```

- JPA는 엔티티(`Payment`) 객체를 영속성 컨텍스트에서 온전히 관리해야 하므로, 엔티티를 반환 타입으로 지정하면 **무조건 테이블의 모든 컬럼을 조회하는 `SELECT *`와 동일한 쿼리**를 날린다.
- 결과적으로 인덱스에 없는 컬럼(`pg_provider`, `card_number` 등)을 채우기 위해 클러스터드 인덱스로 가는 랜덤 I/O가 발생하여 커버링 인덱스의 혜택을 전혀 받지 못한다.

### 2) 해결책: DTO / 인터페이스 프로젝션(Projection)으로 필요한 컬럼만 조회

커버링 인덱스를 태우기 위해 굳이 문자열 결합 위험이 있는 Native Query를 쓸 필요는 없다. JPA가 제공하는 **프로젝션(Projection)** 기능을 사용해 인덱스에 포함된 컬럼만 `SELECT` 절에 명시하면 깔끔하게 해결된다.

#### 방법 ① JPQL `new` 생성자 프로젝션

```java
// 인덱스에 포함된 (userId, paidAt, amount) 3개 컬럼만 DTO로 콕 집어서 조회 -> 커버링 인덱스(Using index) 발동!
@Query("SELECT new com.example.dto.PaymentSummaryDto(p.userId, p.paidAt, p.amount) " +
       "FROM Payment p WHERE p.userId = :userId")
List<PaymentSummaryDto> findSummaryByUserId(@Param("userId") Long userId);
```

#### 방법 ② Spring Data JPA 인터페이스 기반 프로젝션

```java
// 1. 필요한 컬럼의 getter만 정의한 인터페이스 선언
public interface PaymentSummaryView {
    Long getUserId();
    LocalDateTime getPaidAt();
    BigDecimal getAmount();
}

// 2. 리포지토리 반환 타입으로 지정하면 JPA가 해당 3개 컬럼만 SELECT 쿼리로 생성함
List<PaymentSummaryView> findByUserId(Long userId);
```

#### 방법 ③ Querydsl DTO 프로젝션

```java
// 동적 쿼리나 페이징 최적화 시 가장 많이 쓰이는 Querydsl 프로젝션
List<PaymentSummaryDto> result = queryFactory
    .select(Projections.constructor(PaymentSummaryDto.class,
        payment.userId,
        payment.paidAt,
        payment.amount
    ))
    .from(payment)
    .where(payment.userId.eq(userId))
    .fetch();
```

---

## 6. 핵심 요약

1. **커버링 인덱스의 본질**: `WHERE`뿐만 아니라 `SELECT`, `ORDER BY`, `GROUP BY`에 쓰이는 **모든 컬럼이 인덱스 B+Tree 리프 노드에 100% 포함**되어 있어, 실제 테이블 데이터 블록(클러스터드 인덱스)으로 가는 **디스크 랜덤 I/O를 0으로 만드는 기법**이다.
2. **InnoDB의 PK 자동 포함 원리**: InnoDB 세컨더리 인덱스는 리프 노드 끝에 **테이블의 PK를 물리적으로 항상 포함**한다. 따라서 `(user_id)` 인덱스만 있어도 `SELECT id FROM payments WHERE user_id = ?`는 100% 커버링 인덱스로 동작한다.
3. **실행 계획과 JPA 적용**: `EXPLAIN`의 `Extra` 항목에 <strong>`Using index`</strong>가 찍히는지 반드시 확인해야 하며, Spring Data JPA에서는 기본 엔티티 조회가 `SELECT *`를 유발하므로 <strong>인터페이스 프로젝션, JPQL `new` DTO, 또는 Querydsl `Projections`</strong>로 필요한 컬럼만 조회해야 커버링 인덱스가 정상 발동한다.
