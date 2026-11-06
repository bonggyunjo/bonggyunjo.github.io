---
title: "Spring AOP의 등장 배경과 프록시(Proxy) 동작 원리, 그리고 내부 호출(Self-Invocation)의 함정"
date: 2026-11-06T20:40:00+09:00
draft: false
summary: "객체지향 프로그래밍(OOP)의 한계를 보완하기 위해 등장한 AOP(관점 지향 프로그래밍)의 개념과 프록시(Proxy) 객체의 런타임 동작 원리, 그리고 @Transactional 내부 호출(Self-Invocation)이 무효화되는 구조적 원인과 해결 방안을 정리합니다."
categories: ["Backend"]
tags: ["Spring", "Spring Boot", "AOP", "Proxy", "CGLIB", "Transactional", "Self-Invocation", "OOP"]
---

## 1. AOP는 왜 등장했을까? (OOP의 한계와 반복되는 고통)

자바는 대표적인 객체지향 프로그래밍(OOP) 언어다. 우리는 도메인에 따라 역할과 책임을 나누어 아래와 같이 비즈니스 로직을 설계한다.

```text
MemberService (회원 가입, 탈퇴)
OrderService  (주문 생성, 결제 처리)
BoardService  (게시글 작성, 삭제)
```

그런데 서비스 운영 도중 **"서버 성능 모니터링을 위해 모든 서비스의 모든 메서드마다 실행 시간을 측정해서 로그로 남겨라"** 라는 요구사항이 들어왔다고 가정해 보자.

### AOP가 없을 때 마주하는 코드

```java
public void createOrder() {
    long startTime = System.currentTimeMillis(); // 공통 관심사 (부가 기능)

    // === 진짜 핵심 비즈니스 로직 ===
    orderRepository.save(order);
    paymentService.pay(order);
    // ============================

    long endTime = System.currentTimeMillis();   // 공통 관심사 (부가 기능)
    System.out.println("주문 걸린 시간: " + (endTime - startTime) + "ms");
}
```

이 방식은 곧바로 세 가지 심각한 문제를 일으킨다.

1. **극심한 코드 중복**: 서비스 메서드가 100개라면 100군데 모두 똑같은 `startTime`, `endTime` 측정 코드를 복사해 붙여넣어야 한다.
2. **핵심 비즈니스 로직의 오염**: 주문을 처리하는 핵심 코드를 읽어야 하는데 시간 측정, 트랜잭션, 로깅 같은 부가 기능 코드가 뒤섞여 가독성이 크게 떨어진다.
3. **유지보수의 재앙**: 만약 측정 단위를 밀리초(`ms`)에서 나노초(`ns`)로 변경하거나 로그 포맷을 바꾸라는 요청이 오면, 100개의 메서드를 일일이 찾아다니며 수정해야 한다.

---

## 2. AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)란?

**AOP**는 OOP를 대체하는 것이 아니라, OOP만으로는 깔끔하게 분리하기 어려운 <strong>횡단 관심사(Cross-cutting Concern)</strong>를 모듈화하여 OOP를 보완하는 패러다임이다.

소프트웨어 내부의 로직은 크게 두 가지 관심사로 구분할 수 있다.

- **핵심 관심사 (Core Concern)**: 회원 가입, 주문 생성, 결제 처리처럼 비즈니스 고유의 가치를 만드는 핵심 로직
- **공통/횡단 관심사 (Cross-cutting Concern)**: 실행 시간 측정, 트랜잭션 관리(`@Transactional`), 보안 및 권한 체크, 로깅처럼 여러 비즈니스 로직을 가로지르며 반복 등장하는 부가 기능

> **AOP의 핵심 아이디어**
>
> 주문이나 회원 가입 같은 핵심 비즈니스 코드에서는 시간 측정이나 트랜잭션 같은 부가 기능 코드를 전부 걷어내고, 공통 부가 기능은 별도의 모듈(**Aspect**)로 한곳에 모아 정의한 뒤 **스프링이 런타임에 알아서 필요한 위치에 끼워 넣어주도록 만드는 것**이다.

이 덕분에 AOP는 **IoC/DI(제어의 역전/의존성 주입)**, <strong>PSA(서비스 추상화)</strong>와 함께 스프링 프레임워크를 지탱하는 3대 핵심 기둥(스프링 삼각형)을 이룬다.

---

## 3. Spring AOP는 어떻게 동작할까? (프록시 패턴의 원리)

그렇다면 개발자가 `OrderService` 코드 안에 시간 측정이나 트랜잭션 코드를 단 한 줄도 적지 않았는데, 어떻게 실행 전후로 부가 기능이 동작할까?

스프링은 개발자가 작성한 원본 자바 코드를 직접 수정하지 않는다. 대신 **프록시(Proxy, 대리자)** 객체를 생성해 원본 객체를 감싸는 방식을 사용한다.

### 알맹이(Target)와 포장지(Proxy) 구조

- **타겟 객체 (알맹이)**: 개발자가 작성한 순수 비즈니스 로직만 담긴 진짜 `OrderService`
- **프록시 객체 (포장지)**: 스프링이 런타임에 원본 `OrderService`를 상속(또는 인터페이스 구현)받아 겉에 씌워놓은 대리자 객체

스프링 부트가 실행될 때 AOP(`@Transactional`이나 커스텀 `@Aspect`)가 적용된 클래스를 발견하면, 스프링 IoC 컨테이너에는 **진짜 `OrderService` 대신 이를 감싼 프록시 객체가 빈(Bean)으로 등록**된다. 따라서 `OrderController`가 주입받는 `orderService` 역시 실제로는 프록시 객체다.

```text
[컨트롤러 (호출자)]
  │
  ▼ 1. orderService.createOrder() 호출
[AOP 프록시 객체 (가짜 껍데기)]
  │
  ├─> ① 부가 기능 실행: 시간 측정 시작 / 트랜잭션 시작 (beginTransaction)
  │
  ▼ 2. 실제 타겟 객체의 createOrder() 위임 호출
[진짜 OrderService 객체 (알맹이)]
  │
  └─> 순수한 주문 비즈니스 로직만 수행
  │
  ▼ 3. 비즈니스 로직 종료 후 프록시로 복귀
[AOP 프록시 객체 (가짜 껍데기)]
  │
  ├─> ② 부가 기능 마무리: 소요 시간 로그 출력 / 트랜잭션 커밋(commit) 또는 롤백(rollback)
  │
  ▼ 4. 컨트롤러에게 최종 결과 반환
[컨트롤러 (호출자)]
```

### 참고: JDK Dynamic Proxy vs CGLIB
스프링에서 프록시를 생성하는 방식은 두 가지가 있으며, 현재 **Spring Boot 2.x / 3.x는 CGLIB을 기본값**으로 사용한다.
- **JDK Dynamic Proxy**: 자바 리플렉션 기반이며 반드시 **인터페이스**가 있어야만 프록시를 생성할 수 있다.
- **CGLIB (Code Generator Library)**: 바이트코드를 조작해 구체 클래스를 **상속받은 서브클래스 프록시**를 동적으로 생성한다. 인터페이스 없이 구체 클래스만으로도 프록시를 만들 수 있고 런타임 호출 성능이 우수해 기본 전략으로 채택되었다.

---

## 4. 프록시 구조가 만드는 실무 함정: 내부 호출(Self-Invocation) 문제

Spring AOP가 **프록시 객체를 거쳐야만 동작한다**는 원리를 이해하면, 실무에서 가장 자주 겪는 `@Transactional` 무효화 버그인 **내부 호출(Self-Invocation) 문제**의 원인을 정확히 짚어낼 수 있다.

### 문제 상황: 같은 클래스 내부에서 `@Transactional` 메서드를 호출할 때

```java
@Service
public class OrderService {

    // 트랜잭션이 없는 일반 외부 진입점 메서드
    public void processOrder(OrderDto dto) {
        validateOrder(dto);
        
        // 문제 발생 지점: 같은 클래스 내부의 메서드 호출 (this.createOrderAndPay)
        // 프록시 포장지를 거치지 않고 알맹이(this) 내부에서 바로 호출되므로 @Transactional이 완전히 무시됨!
        this.createOrderAndPay(dto);
    }

    @Transactional
    public void createOrderAndPay(OrderDto dto) {
        orderRepository.save(dto.toEntity());
        paymentClient.pay(dto.getAmount()); // 여기서 예외가 터져도 DB 롤백이 되지 않음!
    }
}
```

### 왜 `@Transactional`이 동작하지 않을까?

1. 외부(`Controller`)에서 `orderService.processOrder()`를 호출할 때는 프록시 객체를 통과한다. 하지만 `processOrder()`에는 `@Transactional`이 붙어있지 않으므로 프록시는 트랜잭션을 열지 않고 곧바로 진짜 타겟 객체(`OrderService`)의 `processOrder()`를 호출한다.
2. 일단 진짜 타겟 객체 내부로 들어온 뒤 `this.createOrderAndPay()`를 호출하면, 여기서의 `this`는 스프링 프록시 객체가 아니라 <strong>힙 메모리에 있는 자기 자신(순수 타겟 인스턴스)</strong>을 가리킨다.
3. 결과적으로 프록시 객체를 거치지 않고 내부 메서드로 직행하게 되어, 프록시에 걸려 있는 트랜잭션 시작/커밋/롤백 부가 기능이 전혀 개입하지 못한다.

### 해결 방법: 책임에 따라 별도의 스프링 빈(Service)으로 분리

가장 깔끔하고 권장되는 해결책은 내부 호출이 발생하지 않도록 **트랜잭션 경계가 필요한 로직을 별도의 서비스 빈으로 분리**하여 외부 프록시를 통해 호출되게 만드는 것이다.

```java
@Service
@RequiredArgsConstructor
public class OrderFacade {
    private final OrderCommandService orderCommandService; // 프록시 빈 주입

    public void processOrder(OrderDto dto) {
        validateOrder(dto);
        // 주입받은 외부 빈(프록시)을 통해 호출하므로 @Transactional 정상 작동
        orderCommandService.createOrderAndPay(dto);
    }
}

@Service
public class OrderCommandService {

    @Transactional
    public void createOrderAndPay(OrderDto dto) {
        orderRepository.save(dto.toEntity());
        paymentClient.pay(dto.getAmount());
    }
}
```

---

## 5. 핵심 요약

1. **AOP의 존재 이유**: 시간 측정, 로깅, 트랜잭션처럼 여러 도메인에 걸쳐 반복되는 <strong>횡단 관심사(Cross-cutting Concern)</strong>를 핵심 비즈니스 로직에서 분리해 코드 중복을 없애고 유지보수성을 높인다.
2. **프록시(Proxy) 동작 원리**: 스프링은 원본 타겟 객체를 직접 수정하는 대신, 이를 감싼 <strong>프록시 객체(CGLIB 서브클래스)</strong>를 빈으로 등록해 메서드 호출 전후에 부가 기능을 가로채 실행한다.
3. **내부 호출(Self-Invocation) 주의**: 동일 클래스 내에서 `this.메서드()` 형태로 호출하면 프록시를 거치지 않고 타겟 객체 내부에서 직접 실행되므로 `@Transactional`이나 AOP 어드바이스가 동작하지 않는다. 이 경우 별도의 빈으로 분리해 프록시를 경유하도록 설계해야 한다.
