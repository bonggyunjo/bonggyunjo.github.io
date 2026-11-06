---
title: "객체지향 설계의 5대 원칙(SOLID): 스프링 실무 코드로 이해하는 Bad & Good 패턴"
date: 2026-11-06T20:50:00+09:00
draft: false
summary: "유연하고 변경에 강한 소프트웨어를 설계하기 위한 객체지향 5대 원칙(SRP, OCP, LSP, ISP, DIP)의 핵심 개념과, 실제 스프링 백엔드 코드에서 마주하는 위반 사례(Bad) 및 리팩토링 패턴(Good)을 비교 정리합니다."
categories: ["CS"]
tags: ["OOP", "SOLID", "SRP", "OCP", "LSP", "ISP", "DIP", "Java", "Spring", "Architecture"]
---

## 1. 들어가며: 왜 SOLID 원칙이 필요할까?

코드가 단순히 돌아가는 것과, **요구사항이 바뀔 때 기존 코드를 망가뜨리지 않고 쉽게 고칠 수 있는 것**은 전혀 다른 문제다. 객체지향 설계의 5대 원칙인 **SOLID**는 시간이 지나며 도메인 요구사항이 복잡해지더라도 **응집도는 높이고 결합도는 낮추어** 유지보수와 확장이 쉬운 구조를 만들기 위한 실전 지침이다.

추상적인 문장만으로는 와닿지 않기 쉬운 5가지 원칙을 실제 스프링 백엔드 코드의 <strong>위반 사례(Bad)</strong>와 <strong>개선 사례(Good)</strong>로 대입해 살펴보자.

---

## 2. SRP (단일 책임 원칙, Single Responsibility Principle)

> **"하나의 클래스는 단 하나의 변경 이유(책임)만 가져야 한다."**

여기서 말하는 책임이란 단순한 메서드 개수가 아니라 <strong>변경을 유발하는 주체(원인)</strong>를 뜻한다.

### Bad: 하나의 서비스가 여러 도메인의 잡일을 모두 떠맡은 경우

```java
@Service
public class UserService {

    public void register(UserDto dto) {
        // 1. 회원 정보 검증 및 DB 저장
        userRepository.save(new User(dto));

        // 2. 가입 환영 이메일 전송 (SMTP 직접 통신)
        JavaMailSender mailSender = new JavaMailSenderImpl();
        mailSender.send(message);

        // 3. 신규 가입 웰컴 쿠폰 발급 로직
        couponRepository.save(new Coupon(dto.getUserId()));
    }
}
```

- **문제점**: 이메일 발송 인프라(SMTP ➔ AWS SES)가 바뀌어도 `UserService`를 수정해야 하고, 마케팅팀의 웰컴 쿠폰 정책이 바뀌어도 `UserService`를 수정해야 한다. 즉, 하나의 클래스를 수정해야 하는 이유가 너무 많아 사이드 이펙트가 발생하기 쉽다.

### Good: 변경의 이유에 따라 전담 서비스로 책임을 분리

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final CouponService couponService;

    public void register(UserDto dto) {
        userRepository.save(new User(dto));
        emailService.sendWelcomeEmail(dto);
        couponService.issueWelcomeCoupon(dto.getUserId());
    }
}
```

- **개선 효과**: 메일 발송 방식이 바뀌면 `EmailService`만, 쿠폰 정책이 바뀌면 `CouponService`만 수정하면 되므로 각 클래스의 변경 이유가 단 하나로 명확해진다.

---

## 3. OCP (개방-폐쇄 원칙, Open-Closed Principle)

> **"확장에는 열려 있고(Open), 수정에는 닫혀 있어야(Closed) 한다."**

새로운 기능을 추가할 때 **기존에 잘 동작하던 코드를 뜯어고치지 않고** 새 클래스를 추가하는 것만으로 기능을 확장할 수 있어야 한다는 원칙이다.

### Bad: 새로운 결제 수단이 추가될 때마다 `if-else` 분기문을 뜯어고치는 경우

```java
@Service
public class PaymentService {

    public void pay(String payType, int amount) {
        if ("KAKAO".equals(payType)) {
            // 카카오페이 결제 API 호출
        } else if ("NAVER".equals(payType)) {
            // 네이버페이 결제 API 호출
        } else if ("TOSS".equals(payType)) { 
            // 신규 결제 수단 추가 시 기존 PaymentService 코드를 직접 수정해야 함 (OCP 위반)
        }
    }
}
```

### Good: 인터페이스와 다형성을 활용해 플러그인처럼 확장

```java
// 1. 공통 결제 인터페이스 정의
public interface PaymentGateway {
    boolean supports(String payType);
    void pay(int amount);
}

// 2. 각 결제 수단별로 독립된 구현체 작성
@Component
public class KakaoPayGateway implements PaymentGateway { ... }

@Component
public class TossPayGateway implements PaymentGateway { ... } // 신규 추가 시 클래스만 새로 생성!

// 3. 결제 서비스는 기존 코드를 전혀 수정하지 않음 (Closed for Modification)
@Service
@RequiredArgsConstructor
public class PaymentService {

    // 스프링이 모든 PaymentGateway 구현체 빈을 리스트로 자동 주입
    private final List<PaymentGateway> gateways;

    public void pay(String payType, int amount) {
        PaymentGateway gateway = gateways.stream()
                .filter(g -> g.supports(payType))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("지원하지 않는 결제 수단입니다."));
        gateway.pay(amount);
    }
}
```

- **개선 효과**: 애플페이나 페이코가 새로 추가되더라도 `PaymentGateway`를 구현한 새 클래스만 만들면 될 뿐, 핵심 로직인 `PaymentService` 코드는 단 한 줄도 수정할 필요가 없다.

---

## 4. LSP (리스코프 치환 원칙, Liskov Substitution Principle)

> **"자식(하위) 클래스는 부모(상위) 클래스의 규약과 약속을 깨뜨리지 않고 언제든 부모를 대체할 수 있어야 한다."**

상속 관계에서 부모 타입으로 변수를 선언해 메서드를 호출했을 때, 어떤 자식 객체가 들어오더라도 호출자가 기대한 행위가 일관되게 보장되어야 한다.

### Bad: 부모의 규약을 무시하고 자식 클래스에서 예외를 던지는 경우

```java
public class Account {
    protected int balance;

    // 부모의 약속: 출금을 호출하면 잔액이 차감된다
    public void withdraw(int amount) {
        this.balance -= amount;
    }
}

// 출금 불가(조회/입금 전용) 계좌를 만든다며 무분별하게 Account를 상속받음
public class ReadOnlyAccount extends Account {

    @Override
    public void withdraw(int amount) {
        // 부모의 규약을 깨뜨리고 런타임 예외를 던짐 (LSP 위반)
        throw new UnsupportedOperationException("출금이 불가능한 계좌입니다.");
    }
}
```

- **문제점**: `Account` 타입만 믿고 `account.withdraw(1000)`을 호출한 클라이언트 코드는 `ReadOnlyAccount` 인스턴스가 넘어오는 순간 런타임 에러로 장애를 겪게 된다.

### Good: 역할과 규약에 맞게 계층 및 인터페이스를 명확히 분리

```java
public interface Account {
    int getBalance();
}

// 출금 가능한 계좌만 이 인터페이스를 구현하도록 규약 분리
public interface WithdrawableAccount extends Account {
    void withdraw(int amount);
}
```

- **개선 효과**: `WithdrawableAccount` 타입을 받는 메서드는 어떤 구현체가 들어오더라도 안전하게 `withdraw()`를 호출할 수 있어 부모-자식 간 치환성이 완벽히 보장된다.

---

## 5. ISP (인터페이스 분리 원칙, Interface Segregation Principle)

> **"클라이언트가 자신이 사용하지 않는 메서드에 억지로 의존하거나 구현하도록 강요하지 마라."**

범용적이고 거대한 하나의 인터페이스보다, **역할별로 잘게 쪼갠 여러 개의 구체적인 인터페이스**가 훨씬 낫다는 원칙이다.

### Bad: 비대한 인터페이스 하나에 모든 기능을 몰아넣은 경우

```java
public interface OrderService {
    void createOrder();
    void cancelOrder();
    void refundPayment(); // 환불 처리
    void printInvoice();  // 영수증 인쇄
}

public class SimpleOrderServiceImpl implements OrderService {
    public void createOrder() { ... }
    public void cancelOrder() { ... }

    // 무료/단순 주문이라 환불이나 영수증 인쇄가 필요 없는데도 억지로 구현해야 함 (ISP 위반)
    public void refundPayment() { /* 빈 구현 또는 예외 발생 */ }
    public void printInvoice() { /* 빈 구현 또는 예외 발생 */ }
}
```

### Good: 필요한 기능 단위로 인터페이스를 얇게 분리

```java
public interface OrderManageable {
    void createOrder();
    void cancelOrder();
}

public interface Refundable {
    void refundPayment();
}

public interface InvoicePrintable {
    void printInvoice();
}

// 필요한 인터페이스만 선택하여 깔끔하게 구현
public class SimpleOrderServiceImpl implements OrderManageable {
    public void createOrder() { ... }
    public void cancelOrder() { ... }
}
```

---

## 6. DIP (의존 역전 원칙, Dependency Inversion Principle)

> **"구체적인 구현 클래스에 의존하지 말고, 추상화(인터페이스)에 의존하라."**

변하기 쉬운 구체 클래스(저수준 모듈)에 고수준 비즈니스 로직이 직접 의존하면 하위 기술이 바뀔 때마다 비즈니스 로직까지 연쇄적으로 수정되어야 한다. 이는 스프링 <strong>DI(의존성 주입)와 IoC(제어의 역전)</strong>의 근간이 되는 원칙이다.

### Bad: 서비스가 특정 DB 구현 클래스를 직접 생성(`new`)하여 강하게 결합된 경우

```java
@Service
public class OrderService {

    // 구체 클래스인 MySqlOrderRepository에 직접 의존 (DIP 위반)
    private final MySqlOrderRepository repository = new MySqlOrderRepository();

    public void order() {
        repository.save();
    }
}
```

- **문제점**: 만약 데이터베이스를 MySQL에서 PostgreSQL로 교체하거나 단위 테스트를 위해 가짜(Mock/Fake) 저장소를 끼워 넣으려면 `OrderService` 코드를 직접 뜯어고쳐야 한다.

### Good: 인터페이스(추상화)에만 의존하고 구현체는 스프링으로부터 주입(DI)받음

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    // 구체 클래스가 무엇인지 전혀 알 필요 없이 오직 인터페이스만 바라봄
    private final OrderRepository repository;

    public void order() {
        repository.save();
    }
}
```

- **개선 효과**: 실제 구현체가 `MySqlOrderRepository`이든 `PostgresOrderRepository`이든, 혹은 테스트용 `FakeOrderRepository`이든 관계없이 `OrderService` 코드는 단 한 글자도 수정할 필요가 없다.

---

## 7. 한 줄 핵심 요약

- **SRP (단일 책임 원칙)**: 한 클래스에 너무 많은 역할을 맡기지 말고 변경의 이유를 하나로 유지하라. *(책임 분리)*
- **OCP (개방-폐쇄 원칙)**: 새 기능을 추가할 때 기존 코드를 뜯어고치지 말고 새 클래스를 만들어 끼워 넣어라. *(인터페이스와 다형성)*
- **LSP (리스코프 치환 원칙)**: 부모 타입의 이름표를 달고 나왔다면 부모가 약속한 규약대로 일관되게 행동하라. *(규약 준수)*
- **ISP (인터페이스 분리 원칙)**: 사용하지도 않을 메서드를 억지로 구현하게 강요하지 말고 인터페이스를 작게 쪼개라. *(인터페이스 세분화)*
- **DIP (의존 역전 원칙)**: 구체적인 구현체를 직접 `new` 하지 말고, 인터페이스를 바라보며 외부(스프링 컨테이너)로부터 주입받아라. *(DI / IoC)*
