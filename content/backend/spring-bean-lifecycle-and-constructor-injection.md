---
title: "Spring Bean 생명주기 콜백과 생성자 주입(Constructor Injection)의 불변성·순환참조 방지 원리"
date: 2026-11-09T22:50:00+09:00
draft: false
summary: "서버가 구동되다 죽는 이유를 파악하고 자원 누수를 막으며 안전한 애플리케이션 구조를 설계하기 위해 반드시 알아야 할 Spring Bean 생명주기(@PostConstruct, @PreDestroy, BeanPostProcessor)와 생성자 주입의 동작 원리를 정리합니다."
categories: ["Backend"]
tags: ["Spring", "Spring Boot", "Bean Lifecycle", "DI", "Constructor Injection", "Circular Dependency", "PostConstruct", "BeanPostProcessor"]
---

## 1. 들어가며: 왜 빈 생명주기와 의존성 주입 원리를 알아야 할까?

스프링 부트로 개발하다 보면 `@Service`, `@RequiredArgsConstructor`, `@PostConstruct` 같은 애노테이션을 습관처럼 사용한다. 하지만 그 내부의 동작 순서와 원리를 정확히 이해해야 하는 이유는 명확하다.

> **서버가 구동되다 죽는 이유를 파악하고, 자원 누수를 막으며, 안전한 애플리케이션 구조를 설계하기 위한 실무적인 필수 지식이기 때문이다.**

실무에서 이 원리를 모르면 다음과 같은 문제를 마주하게 된다.

1. **서버 기동 실패와 초기화 시점 버그**: 의존성 주입이 완료되기도 전에 초기화 로직이 실행되어 `NullPointerException`이 발생하거나, 빈 간 순환 참조(`BeanCurrentlyInCreationException`)로 인해 배포 중 서버가 죽는 원인을 즉시 파악해야 한다.
2. **외부 자원 누수(Resource Leak) 방지**: 외부 소켓 연결, 스레드 풀, 캐시 워밍업처럼 애플리케이션 시작 시 안전하게 열고 종료 시 누수 없이 반납(`Graceful Shutdown`)해야 하는 자원의 정확한 개입 시점을 알아야 한다.
3. **런타임 시한폭탄 제거와 불변 구조 설계**: 왜 실무에서 `@Autowired` 필드 주입을 금지하고 <strong>생성자 주입(Constructor Injection)</strong>을 강제하는지, 그것이 어떻게 런타임 장애(`StackOverflowError`)를 기동 시점(Fail-Fast)으로 끌어당겨 막아주는지 구조적으로 이해해야 한다.

---

## 2. Spring Bean 생명주기(Lifecycle)와 콜백 메커니즘

스프링 IoC 컨테이너가 관리하는 빈(Bean)은 단순히 `new` 키워드로 생성되고 끝나는 것이 아니라, 컨테이너의 시작부터 종료까지 명확한 생명주기를 거친다.

### 1) 스프링 빈의 전체 생명주기 흐름

```text
[스프링 IoC 컨테이너 생성]
   │
   ▼
[1. 스프링 빈 인스턴스화 (객체 생성)]
   │  └─ 생성자 주입(Constructor Injection)은 이 단계에서 객체 생성과 의존관계 주입이 동시에 일어남
   ▼
[2. 의존관계 주입 (DI - 필드 주입 / 수정자 주입)]
   │  └─ @Autowired 필드나 Setter를 통한 주입은 빈 껍데기 생성 이후 이 단계에서 수행됨
   ▼
[3. 초기화 전 빈 후처리기 (BeanPostProcessor.postProcessBeforeInitialization)]
   │
   ▼
[4. 초기화 콜백 (@PostConstruct / InitializingBean.afterPropertiesSet)]
   │  └─ 모든 의존관계 주입이 끝난 것이 보장된 안전한 초기화 시점
   ▼
[5. 초기화 후 빈 후처리기 (BeanPostProcessor.postProcessAfterInitialization)]
   │  └─ ★ 핵심: @Transactional 등 Spring AOP 프록시(Proxy) 객체로 감싸져 바꿔치기되는 단계
   ▼
[6. 빈 사용 (애플리케이션 런타임 동작)]
   │
   ▼
[7. 소멸 전 콜백 (@PreDestroy / DisposableBean.destroy)]
   │  └─ 컨테이너가 종료되기 직전 외부 커넥션, 스레드 풀 등 리소스를 안전하게 반납하는 시점
   ▼
[스프링 컨테이너 종료]
```

### 2) 왜 '객체 생성'과 '초기화(`@PostConstruct`)'를 분리해야 할까?

자바 객체의 생성자(Constructor)는 필수 필드 값을 받아 **메모리에 인스턴스를 할당하는 일**에만 집중해야 한다. 반면 외부 네트워크 소켓을 연결하거나 DB에서 초기 캐시 데이터를 퍼 올리는 무거운 작업은 생성자가 아닌 **`@PostConstruct` 초기화 콜백**에서 수행하는 것이 원칙이다.

그 이유는 두 가지다.

1. **의존관계 주입 미완료 위험 차단**: 만약 필드 주입이나 세터 주입을 사용하는 빈에서 생성자 내부에 초기화 로직을 작성하면, 아직 `@Autowired` 필드가 주입되기 전(`null` 상태)이므로 서버가 기동되자마자 `NullPointerException`을 뿜으며 죽는다.
2. **단일 책임 원칙(SRP)과 자원 누수 방지**: 객체 생성 자체와 외부 리소스 연결을 분리해야 단위 테스트에서 객체를 가볍게 생성할 수 있다. 또한 `@PostConstruct`로 연 외부 리소스는 짝을 이루는 <strong>`@PreDestroy`</strong>에서 닫아주어야 배포나 재기동 시 커넥션 누수(Leak)를 막을 수 있다.

```java
@Component
@RequiredArgsConstructor
public class ExternalSocketClient {

    private final SocketConfig socketConfig; // 1. 생성자 주입으로 필수 의존성 확보
    private SocketConnection connection;

    // 2. 의존성 주입이 100% 완료된 후 안전하게 외부 연결 오픈
    @PostConstruct
    public void init() {
        this.connection = SocketConnection.open(socketConfig.getHost(), socketConfig.getPort());
    }

    // 3. 서버(컨테이너)가 내려가기 직전에 열린 연결을 안전하게 반환하여 자원 누수 방지
    @PreDestroy
    public void close() {
        if (this.connection != null && this.connection.isOpen()) {
            this.connection.close();
        }
    }
}
```

### 3) 실무 주의점: `@PostConstruct`와 AOP 프록시(`@Transactional`)의 시점 차이

위 생명주기 흐름도에서 <strong>4번(`@PostConstruct`)</strong>과 <strong>5번(`BeanPostProcessor` 후처리)</strong>의 순서를 눈여겨보아야 한다.

- `@PostConstruct`는 빈의 초기화 단계(4번)에서 실행된다.
- 반면 `@Transactional`이나 `@Async` 같은 **Spring AOP 프록시 객체 생성**은 초기화가 끝난 직후인 <strong>빈 후처리기(`postProcessAfterInitialization`, 5번)</strong> 단계에서 완료된다.

따라서 `@PostConstruct` 메서드 내부에서 자기 자신의 `@Transactional` 메서드를 호출하거나 프록시 기반 부가 기능에 의존하면, **아직 프록시 포장지가 씌워지기 전인 원본 객체 상태에서 실행되어 트랜잭션이 적용되지 않는 문제**가 발생할 수 있다. 모든 빈의 프록시 생성까지 완전히 끝난 시점에 초기화 로직을 돌리고 싶다면 `@EventListener(ApplicationReadyEvent.class)`를 사용하는 것이 안전하다.

---

## 3. 생성자 주입(Constructor Injection)을 필수로 사용해야 하는 이유

스프링이 지원하는 의존성 주입 방식에는 크게 **필드 주입(`@Autowired`)**, **수정자(Setter) 주입**, **생성자(Constructor) 주입** 세 가지가 있다. 과거에는 코드가 짧다는 이유로 `@Autowired` 필드 주입이 흔히 쓰였지만, 현재 스프링 공식 문서와 실무 표준은 모두 **생성자 주입**을 강력히 권장한다.

단순히 "권장하니까"가 아니라, 생성자 주입이 <strong>불변성(Immutability)</strong>과 **순환 참조(Circular Dependency) 사전 차단**이라는 구조적 안전장치를 어떻게 제공하는지 비교해 보자.

### 1) 객체의 불변성(Immutability) 보장 (`final` 키워드)

- **필드 주입 / 수정자 주입의 한계**: 빈을 먼저 기본 생성자로 만든 뒤 나중에 값을 주입하는 방식이므로 필드에 `final` 키워드를 붙일 수 없다. 즉, 런타임 도중 누군가 실수로 의존 객체를 `null`이나 다른 객체로 바꿔치기할 위험에 열려 있다.
- **생성자 주입의 강점**: 객체가 생성되는 시점에 딱 한 번만 의존성이 주입되므로 필드를 <strong>`private final`</strong>로 선언할 수 있다.
  - 한 번 주입된 의존 객체는 애플리케이션이 종료될 때까지 절대 변경되지 않는다(불변성 보장).
  - 혹시라도 개발자가 의존성 주입 코드를 누락하면 **컴파일 시점**(`variable might not have been initialized`)에 즉시 에러를 잡아준다.

### 2) 순환 참조(Circular Dependency)의 조기 차단 원리

실무에서 도메인이 복잡해지면 `OrderService`가 `PaymentService`를 참조하고, 동시에 `PaymentService`도 `OrderService`를 참조하는 **순환 참조** 실수가 종종 발생한다. 이때 필드 주입과 생성자 주입은 완전히 다른 결과를 낳는다.

#### Bad Pattern: 필드 주입(`@Autowired`) 사용 시의 참사

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService; // 1. 빈 껍데기 생성 후 나중에 주입

    public void createOrder() {
        paymentService.pay();
    }
}

@Service
public class PaymentService {
    @Autowired
    private OrderService orderService; // 1. 빈 껍데기 생성 후 나중에 주입

    public void pay() {
        orderService.createOrder();
    }
}
```

- **동작 과정**: 필드 주입은 <strong>"일단 기본 생성자로 빈 껍데기 인스턴스를 먼저 다 만들어 둔 뒤, 리플렉션으로 필드를 연결"</strong>한다.
- `OrderService` 껍데기 생성 성공 -> `PaymentService` 껍데기 생성 성공 -> 서로의 필드에 참조 연결 완료.
- **치명적 결과**: 스프링 부트 서버가 **아무런 에러 없이 정상 기동**되어 버린다(스프링 부트 2.6 이상에서는 기본적으로 순환 참조를 막지만, 필드 주입의 구조적 메커니즘 자체가 객체 생성과 주입이 분리되어 있어 생성 시점에 이를 강제하지 못한다). 만약 이 상태로 운영 서버에 배포되면, 실제 사용자가 결제 버튼을 눌러 **메서드가 호출되는 런타임 순간에 무한 재귀 호출이 일어나며 `StackOverflowError`로 서버가 죽는다.**

#### Good Pattern: 생성자 주입(`@RequiredArgsConstructor`) 사용 시의 방어

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentService paymentService; // 생성자 파라미터로 필수 요구
}

@Service
@RequiredArgsConstructor
public class PaymentService {
    private final OrderService orderService; // 생성자 파라미터로 필수 요구
}
```

- **동작 과정**: 생성자 주입은 **"객체를 생성(`new`)하는 바로 그 순간에 파라미터로 의존 빈이 먼저 완성되어 들어와야"** 한다.
  1. 컨테이너가 `new OrderService(paymentService)`를 호출하려 보니 `PaymentService` 빈이 필요하다.
  2. 그래서 `new PaymentService(orderService)`를 먼저 만들려 보니 다시 `OrderService` 빈이 필요하다.
  3. 서로가 서로의 완성품을 먼저 내놓으라고 요구하므로 **객체 생성 단계에서 즉각 교착 상태**가 드러난다.
- **안전한 결과 (Fail-Fast)**: 서버가 켜지는 도중 즉시 <strong>`BeanCurrentlyInCreationException`</strong>을 던지며 **기동 자체가 실패**한다. 덕분에 잘못된 코드가 운영 환경에 배포되어 런타임 장애를 일으키는 일을 원천 차단할 수 있다.

> **비유로 이해하기**
>
> 필수 부품들이 완전히 맞물려 조립되어야만 완성품을 책상 위에 올려놓는 것이 **생성자 주입**이라면, 빈 껍데기만 먼저 올려놓고 나중에 부품을 끼워 넣으려다 서로가 서로를 기다리며 무너지는 것이 **필드 주입의 순환 참조**다.

### 3) 순수 자바(POJO) 단위 테스트의 용이성

`@Autowired` 필드 주입으로 작성된 클래스는 스프링 컨테이너의 리플렉션 없이는 외부에서 의존 객체를 넣어줄 방법이 없다. 결국 단순한 비즈니스 로직 하나를 테스트하려 해도 무거운 `@SpringBootTest`를 띄우거나 리플렉션 유틸리티를 동원해야 한다.

반면 생성자 주입으로 설계된 클래스는 스프링 컨테이너 없이도 순수 자바 코드만으로 즉시 단위 테스트를 작성할 수 있다.

```java
@Test
void 주문_생성_단위_테스트() {
    // 스프링 컨텍스트를 띄울 필요 없이 new 키워드로 가짜(Mock/Fake) 객체를 직접 주입 가능!
    PaymentService mockPayment = new FakePaymentService();
    OrderService orderService = new OrderService(mockPayment);

    orderService.createOrder();
}
```

---

## 4. 순환 참조가 발생했을 때의 올바른 해결법

생성자 주입 덕분에 기동 시점에 `BeanCurrentlyInCreationException`(순환 참조)을 발견했다면, 이를 어떻게 해결해야 할까?

`@Lazy` 애노테이션을 붙여 프록시 지연 로딩으로 에러만 덮는 것은 임시방편일 뿐이다. **두 서비스가 서로를 참조한다는 것은 객체 간의 책임 분리(SRP)와 의존 방향 설계가 잘못되었다는 신호**이므로, 구조적으로 결합을 끊어야 한다.

1. **공통 역할 추출 또는 상위 퍼사드(Facade) 분리**: `OrderService`와 `PaymentService`가 서로를 호출해야 했던 공통 흐름을 상위 `OrderPaymentFacade`로 끌어올려 단방향(`Facade -> OrderService, PaymentService`)으로만 흐르게 재설계한다.
2. **스프링 이벤트(`ApplicationEventPublisher`) 기반 비결합 처리**: 결제 완료 후 주문 상태를 변경해야 한다면 `PaymentService`가 `OrderService`를 직접 주입받아 호출하는 대신, `PaymentCompletedEvent`를 발행하고 `OrderEventListener`가 이를 받아 처리하도록 만들어 양방향 참조를 끊어낸다.

---

## 5. 핵심 요약

| 구분 | 핵심 내용 |
| :--- | :--- |
| **빈 생명주기 콜백** | 스프링 빈은 **객체 생성 -> 의존관계 주입 -> 초기화 콜백(`@PostConstruct`) -> 사용 -> 소멸 콜백(`@PreDestroy`)** 순으로 동작하며, 외부 자원 연결과 해제는 생성자가 아닌 생명주기 콜백에서 처리해야 `NullPointerException`과 자원 누수를 막을 수 있다. |
| **초기화와 AOP 시점** | `@PostConstruct`는 `BeanPostProcessor`가 AOP 프록시를 씌우기 전에 실행되므로, 초기화 시점에 프록시 기능(`@Transactional` 등)이 필요하다면 `ApplicationReadyEvent` 시점을 활용해야 한다. |
| **생성자 주입의 필수성** | 1) `final` 키워드를 통한 **불변성 보장**, 2) 객체 생성 시점에 의존성을 강제하여 **순환 참조를 서버 기동 단계(Fail-Fast)에서 즉시 차단**, 3) 스프링 없이도 `new` 키워드로 검증 가능한 **순수 POJO 단위 테스트 지원**이라는 세 가지 결정적 이점을 제공한다. |
