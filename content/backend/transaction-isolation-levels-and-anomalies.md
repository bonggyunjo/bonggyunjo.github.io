---
title: "트랜잭션 격리 수준(Isolation Level) 4단계와 3대 이상현상(Anomalies)의 원리"
date: 2026-11-23T13:25:00+09:00
draft: false
summary: "동시성과 데이터 정합성 사이의 트레이드오프를 해결하기 위한 ACID와 직렬 가능성(Serializability), 2PL vs MVCC 엔진 제어 원리, 3대 이상현상(Dirty Read, Non-Repeatable Read, Phantom Read), 그리고 ANSI SQL 4단계 격리 수준과 MySQL InnoDB의 동작 방식을 정리합니다."
categories: ["Backend"]
tags: ["Database", "Transaction", "Isolation Level", "MVCC", "2PL", "MySQL", "InnoDB", "ACID"]
---

## 1. ACID, 인터리빙, 그리고 직렬 가능성(Serializability)

트랜잭션 격리 수준과 3대 이상현상은 결국 데이터베이스의 근본적인 딜레마에서 출발한다.

> **"여러 작업을 동시에 처리해서 성능을 높이면서도, 마치 한 번에 하나씩 실행한 것처럼 데이터를 안전하게 지킬 수 있을까?"**

데이터베이스에서 <strong>트랜잭션(Transaction)</strong>은 하나의 논리적 작업 단위이며, 안전하게 동작하기 위해 다음 4가지 성질(**ACID**)을 보장해야 한다.

1. **원자성 (Atomicity)**: 트랜잭션 내 작업은 모두 성공하거나 모두 실패해야 한다.
2. **일관성 (Consistency)**: 트랜잭션 수행 전후에 걸쳐 DB의 제약조건과 규칙이 일관되게 유지되어야 한다.
3. **격리성 (Isolation)**: 동시에 실행되는 트랜잭션들끼리 서로 간섭하지 않아야 한다.
4. **지속성/영속성 (Durability)**: 커밋이 완료된 트랜잭션의 결과는 시스템 장애가 발생하더라도 영구히 저장되어야 한다.

### 인터리빙(Interleaving)과 직렬 가능성

성능을 높이기 위해 DB는 여러 트랜잭션의 읽기·쓰기 연산을 번갈아 실행하는데, 이를 <strong>인터리빙(Interleaving)</strong>이라 하며 그 실행 순서를 스케줄(Schedule)이라고 부른다.

- **직렬 스케줄 (Serial Schedule)**: 트랜잭션 A가 완전히 끝난 뒤에야 트랜잭션 B를 실행한다. 100% 안전하지만 동시 처리량이 떨어져 느리다.
- **직렬 가능성 (Serializability)**: 연산을 뒤섞어 동시에 실행(Concurrent Schedule)했더라도, **최종 결과만 놓고 보면 직렬 스케줄로 실행한 것과 완전히 동일**하다면 안전하다(Serializable)고 판단하는 이론적 기준이다.

여러 트랜잭션을 동시에 실행하다가 바로 이 **직렬성이 깨질 때 발생하는 대표적인 부작용**들을 유형별로 묶어 놓은 것이 **3대 이상현상**이다.

---

## 2. DB 엔진의 동시성 제어 메커니즘: 2PL vs MVCC

격리 수준이 *"무엇을 막아야 하는지"*를 정한 규칙이라면, DB 엔진이 이를 실제로 구현하는 핵심 메커니즘은 크게 두 가지로 나뉜다.

### 1) 락 기반 제어 — 2PL (Two-Phase Locking)

- 데이터를 읽을 때 공유 락(`S-Lock`), 쓸 때 배타 락(`X-Lock`)을 걸어 충돌을 원천 차단하는 방식이다.
- 직관적이지만 읽기 작업과 쓰기 작업이 서로를 대기(Blocking)하게 만들어 성능 저하나 데드락(Deadlock)이 발생하기 쉽다.

### 2) 버전 기반 제어 — MVCC (Multi-Version Concurrency Control)

- 데이터를 수정할 때 기존 값을 덮어쓰지 않고 **Undo Log** 등에 이전 버전을 남겨둔다.
- 읽기 트랜잭션은 락을 걸지 않고 자신에게 맞는 시점의 버전(Snapshot)을 읽으므로 **읽기와 쓰기가 서로를 막지 않는다.** MySQL InnoDB, PostgreSQL 등 현대 RDBMS의 핵심 동작 원리다.

---

## 3. 3대 이상현상(Anomalies)의 원리

동시에 실행되는 두 트랜잭션(`Tx A`, `Tx B`)이 얽힐 때 발생하는 대표적인 데이터 불일치 문제 세 가지는 다음과 같다.

### 1) Dirty Read (오손 읽기)
- **원리**: `Tx B`가 데이터를 수정하고 **아직 커밋(Commit)하지 않은 상태**인데, `Tx A`가 그 변경된 값을 먼저 읽어버리는 현상이다.
- **문제점**: 이후 `Tx B`가 에러 등으로 롤백(Rollback)하면, `Tx A`는 DB에 존재한 적도 없는 가짜 데이터를 기준으로 비즈니스 로직을 처리하게 된다.

### 2) Non-Repeatable Read (반복 불가능한 읽기)
- **원리**: `Tx A`가 같은 조건으로 특정 행(Row)을 두 번 조회했는데, 그사이에 `Tx B`가 해당 행을 **수정(`UPDATE`)하고 커밋**하여 첫 번째 조회 결과와 두 번째 조회 결과가 달라지는 현상이다.
- **문제점**: 하나의 트랜잭션 안에서 같은 데이터를 읽었는데도 값이 바뀌어 일관성이 깨진다.

### 3) Phantom Read (유령 읽기)
- **원리**: `Tx A`가 특정 범위 조건(`WHERE age >= 20` 등)으로 데이터를 두 번 조회했는데, 그사이에 `Tx B`가 조건에 맞는 새로운 행을 **삽입(`INSERT`)하거나 삭제(`DELETE`)하고 커밋**하여, 없던 행이 갑자기 나타나거나 있던 행이 사라지는 현상이다.
- **Non-Repeatable Read와의 차이점**: `Non-Repeatable Read`는 **기존 단일 행의 값 변경**이 원인이고, `Phantom Read`는 **범위 내 행의 개수(집합) 변화**가 원인이다.

---

## 4. 트랜잭션 격리 수준(Isolation Level) 4단계

<strong>트랜잭션 격리 수준(Isolation Level)</strong>은 여러 트랜잭션이 동시에 같은 데이터에 접근할 때 서로의 작업에 어느 정도까지 영향을 미치지 않도록 차단할지 결정하는 기준이다.

격리 수준이 높을수록 **데이터 정합성**은 강해지지만, 락 경합이나 버전 관리 오버헤드가 늘어 **동시성 성능**은 떨어지는 트레이드오프 관계에 있다. ANSI/ISO SQL 표준은 위 3대 이상현상을 허용하는 정도에 따라 4단계를 정의한다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read | 동시성 / 정합성 및 특징 |
| :--- | :--- | :--- | :--- | :--- |
| **1. READ UNCOMMITTED** | 발생 | 발생 | 발생 | 동시성 최고 / 정합성 최저 |
| **2. READ COMMITTED** | 방지 | 발생 | 발생 | PostgreSQL, Oracle 등 대부분 RDBMS 기본값 |
| **3. REPEATABLE READ** | 방지 | 방지 | 발생 (표준 기준) | MySQL / MariaDB(InnoDB) 기본값 |
| **4. SERIALIZABLE** | 방지 | 방지 | 방지 | 동시성 최저 / 정합성 최고 |

### 단계별 동작 원리

1. **`READ UNCOMMITTED`**
   - 커밋 여부와 상관없이 다른 트랜잭션의 변경 내용을 즉시 읽는다. 정합성 보장이 거의 안 되어 일반적인 서비스에서는 거의 사용하지 않는다.
2. **`READ COMMITTED`**
   - **커밋이 완료된 데이터만** 읽을 수 있다. 커밋 전의 변경 내용은 언두 로그(Undo Log) 등에서 이전 버전을 읽어 `Dirty Read`를 막는다. 다만 같은 트랜잭션 안에서도 조회 시점마다 최신 커밋 본을 읽으므로 `Non-Repeatable Read`가 발생할 수 있다.
3. **`REPEATABLE READ`**
   - 트랜잭션이 시작될 때의 스냅샷(Snapshot)을 기준으로 데이터를 읽는다. 다른 트랜잭션이 값을 수정하고 커밋해도 내 트랜잭션은 시작 시점의 버전을 계속 읽으므로 `Non-Repeatable Read`를 방지한다.
   - **MySQL InnoDB의 특징**: InnoDB는 기본 격리 수준이 `REPEATABLE READ`이며, MVCC 스냅샷과 갭 락(Gap Lock)을 함께 활용해 SQL 표준과 달리 일반적인 조회/잠금 조회에서도 `Phantom Read`까지 방어한다.
4. **`SERIALIZABLE`**
   - 모든 트랜잭션을 사실상 순차적으로 실행하는 것과 동일한 결과를 보장한다. 일반 읽기 작업에도 공유 락(`S-Lock`)이 걸리는 등 제약이 강해 동시 처리량이 크게 떨어진다.
