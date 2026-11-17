---
title: "@Transactional 전파 속성(Propagation): REQUIRED vs REQUIRES_NEW의 트랜잭션 롤백 격리 원리"
date: 2026-11-17T19:15:00+09:00
draft: false
summary: "트랜잭션의 기본 개념부터 전파 속성(Propagation)의 역할, 물리 트랜잭션과 논리 트랜잭션의 구분, 그리고 Spring @Transactional의 REQUIRED와 REQUIRES_NEW가 커넥션 공유 및 롤백 격리를 어떻게 처리하는지 정리합니다."
categories: ["Backend"]
tags: ["Spring", "Spring Boot", "Transactional", "Propagation", "REQUIRED", "REQUIRES_NEW", "Database", "Backend"]
---

## 1. 트랜잭션의 개념과 전파 속성(Propagation)이란?

### 트랜잭션(Transaction)이란?

**트랜잭션은 데이터베이스에서 하나의 논리적 기능을 수행하기 위해 묶은 작업의 최소 단위**다. 여러 개의 작업이 **모두 성공(Commit)하거나, 하나라도 실패하면 전부 취소(Rollback)되어야 하는 하나의 묶음**으로 동작한다.

가장 대표적인 비유가 **계좌 이체**다.

```text
A가 B에게 1만 원을 송금할 때 DB에서는 두 가지 작업이 일어난다.
1. A의 계좌에서 1만 원 차감 (UPDATE)
2. B의 계좌에 1만 원 증가 (UPDATE)

만약 1번은 성공했는데 갑자기 서버가 다운되어 2번이 실패한다면?
➔ A의 돈 1만 원만 허공으로 사라지는 심각한 데이터 정합성 사고가 발생한다.
```

이를 막기 위해 1번과 2번을 하나의 트랜잭션으로 묶어 두면, 둘 다 완벽히 성공했을 때만 최종 반영(`Commit`)하고 중간에 하나라도 실패하면 처음부터 송금을 안 한 상태(`Rollback`)로 되돌린다.

### 전파 속성(Propagation)이란?

그렇다면 <strong>전파 속성(Propagation)</strong>은 무엇일까?

> <strong>"트랜잭션이 이미 진행 중일 때, 또 다른 `@Transactional` 메서드를 만나면 어떻게 행동할 것인가?"</strong>를 결정하는 규칙이다.

스프링의 `@Transactional` 기본 전파 속성은 <strong>`REQUIRED`</strong>이며, 실무에서 가장 자주 비교되는 <strong>`REQUIRED`</strong>와 <strong>`REQUIRES_NEW`</strong>의 핵심 차이는 **기존 트랜잭션에 참여하는가, 아니면 완전히 독립적인 새 물리 트랜잭션을 생성하는가**에 있다.

---

## 2. 물리 트랜잭션 vs 논리 트랜잭션

전파 속성을 이해하려면 먼저 스프링이 구분하는 **물리 트랜잭션**과 **논리 트랜잭션**의 차이를 알아야 한다.

- **물리 트랜잭션 (Physical Transaction)**: 실제 DB와 연결되어 커밋/롤백을 수행하는 진짜 **DB 커넥션 1개** 단위
- **논리 트랜잭션 (Logical Transaction)**: 코드상에서 부모 메서드, 자식 메서드 각각에 `@Transactional`이 붙어 있는 가상의 트랜잭션 단위

```text
[물리 트랜잭션: DB 커넥션 딱 1개 공유]
┌────────────────────────────────────────────────────────┐
│  [논리 트랜잭션 1: 부모 (order)]                         │
│  orderRepository.save(order);                          │
│                                                        │
│     ┌────────────────────────────────────────────┐     │
│     │  [논리 트랜잭션 2: 자식 (paymentLog)]         │     │
│     │  logRepository.save(log);                  │     │
│     └────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘
```

스프링은 여러 개의 **논리 트랜잭션**을 하나의 **물리 트랜잭션**으로 묶을 수도 있고(`REQUIRED`), 각각 별도의 물리 트랜잭션으로 쪼갤 수도 있다(`REQUIRES_NEW`).

---

## 3. `Propagation.REQUIRED` (기본값)

**`REQUIRED`는 진행 중인 트랜잭션이 있으면 그 트랜잭션에 참여(합류)하고, 없으면 새로운 트랜잭션을 생성하는 방식**이다.

- 바깥 메서드(부모)와 안쪽 메서드(자식)가 **하나의 물리 트랜잭션(DB 커넥션 1개)을 공유**한다.
- 계좌 이체나 일반 주문 처리처럼 여러 작업이 **하나의 원자적(Atomic) 단위**로 묶여야 하는 대부분의 비즈니스 로직에 사용한다.

### 롤백 동작 원리

하나의 물리 트랜잭션 안에서 처리되므로, 어느 한쪽의 논리 트랜잭션에서 `Unchecked Exception`(런타임 예외)이 발생하면 **모든 작업이 함께 롤백**된다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentLogService paymentLogService;

    @Transactional // 기본값: REQUIRED (물리 트랜잭션 시작)
    public void order(Order order) {
        orderRepository.save(order);
        try {
            paymentLogService.saveLog(order.getId()); // 기존 트랜잭션에 합류
        } catch (RuntimeException e) {
            // 주의: 여기서 예외를 try-catch로 삼켜도,
            // 자식 논리 트랜잭션이 이미 rollback-only 마크를 찍어두었기 때문에
            // 최종 커밋 시점에 UnexpectedRollbackException이 발생하며 전체 롤백된다!
        }
    }
}
```

특히 주의할 점은 자식 메서드에서 런타임 예외가 터졌을 때 부모가 `try-catch`로 예외를 잡더라도, **물리 트랜잭션을 공유하는 구조상 트랜잭션 동기화 매니저에 `rollback-only` 마크가 남는다**는 것이다. 결국 부모 메서드가 끝나고 커밋을 시도하는 순간 `UnexpectedRollbackException`이 발생하며 전체가 롤백된다.

---

## 4. `Propagation.REQUIRES_NEW`

**`REQUIRES_NEW`는 진행 중인 트랜잭션이 있든 없든 상관없이 무조건 새로운 물리 트랜잭션을 생성하는 방식**이다.

```text
[물리 트랜잭션 A: DB 커넥션 1번] (잠시 보류 Suspend)
  └─ orderRepository.save(order);
        │
        ▼ (REQUIRES_NEW 호출: 새 커넥션 획득)
[물리 트랜잭션 B: DB 커넥션 2번] (완전히 독립적으로 실행 후 커밋/롤백)
  └─ logRepository.save(log);
```

- 외부에서 이미 트랜잭션이 시작되었더라도 그 트랜잭션을 잠시 보류(`Suspend`)해 두고, **별도의 DB 커넥션을 새로 얻어 독립적인 물리 트랜잭션을 시작**한다.
- 따라서 자식 트랜잭션 내부에서 예외가 발생해 롤백되더라도 외부 트랜잭션에 영향을 주지 않고 내부만 따로 롤백할 수 있으며, 반대로 메인 비즈니스 로직(외부)이 실패해 롤백되더라도 **자식 트랜잭션에서 남긴 데이터는 커밋되어 보존**된다.
- **주요 사용 사례**: 주문이나 결제가 실패하더라도 **반드시 DB에 기록되어야 하는 감사 로그(Audit Log), 결제 실패 이력, 에러 로그 저장** 등에 활용한다.

```java
@Service
@RequiredArgsConstructor
public class PaymentLogService {
    private final LogRepository logRepository;

    // 부모 트랜잭션의 롤백 여부와 무관하게 독립된 물리 트랜잭션으로 커밋/롤백
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(Long orderId) {
        logRepository.save(new PaymentLog(orderId));
    }
}
```

### 참고: `REQUIRES_NEW` 사용 시 커넥션 풀(HikariCP) 주의점

`REQUIRES_NEW`는 기존 트랜잭션이 DB 커넥션 1개를 쥔 채로 보류 상태에서 새 커넥션 1개를 추가로 요구한다. 즉, **스레드 1개가 DB 커넥션 2개를 동시에 점유**하게 된다. 만약 커넥션 풀(HikariCP) 크기가 작은 상태에서 동시 요청이 몰리면, 모든 스레드가 1번 커넥션을 쥔 채 2번 커넥션을 기다리며 <strong>커넥션 풀 고갈(Deadlock)</strong>이 발생할 수 있으므로 꼭 필요한 분리 지점에만 신중하게 사용해야 한다.

---

## 5. `REQUIRED` vs `REQUIRES_NEW` 한눈에 비교

| 구분 | `Propagation.REQUIRED` (기본값) | `Propagation.REQUIRES_NEW` |
| :--- | :--- | :--- |
| **트랜잭션 참여** | 기존 트랜잭션이 있으면 **참여** (없으면 새로 생성) | 기존 트랜잭션 여부와 무관하게 **항상 새 트랜잭션 생성** |
| **DB 커넥션** | 부모·자식이 **물리 커넥션 1개 공유** | 기존 커넥션 보류 후 **별도의 커넥션 1개 추가 사용** |
| **롤백 범위** | 한쪽에서 실패하면 **전체(부모+자식) 동반 롤백** | 부모와 자식의 **커밋/롤백이 완전히 독립 격리**됨 |
| **대표 활용처** | 계좌 이체, 주문+재고 차감 등 하나의 원자적 작업 | 결제 실패/감사 로그, 이력 기록 등 반드시 남아야 하는 작업 |
