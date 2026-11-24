---
title: "JPA 복합키 매핑: @IdClass vs @EmbeddedId의 차이와 equals & hashCode가 필수인 이유"
date: 2026-11-24T12:20:00+09:00
draft: false
summary: "RDBMS의 복합키를 JPA에서 다루기 위해 별도의 식별자 클래스가 필요한 이유부터 @IdClass와 @EmbeddedId의 구조적 차이, 1차 캐시와 HashMap 원리로 이해하는 equals·hashCode 오버라이딩, 그리고 대리키 선호 이유와 Persistable을 활용한 save() 최적화까지 알기 쉽게 정리합니다."
categories: ["Backend"]
tags: ["JPA", "Hibernate", "Composite Key", "IdClass", "EmbeddedId", "Persistable", "Spring Data JPA", "Backend"]
---

## 1. 왜 별도의 '식별자 클래스'가 필요할까?

관계형 데이터베이스(RDB)에서는 두 개 이상의 컬럼을 묶어 기본키(PK)로 사용하는 <strong>복합키(Composite Key)</strong>가 흔하다. 하지만 JPA에서 복합키를 매핑하려면 반드시 별도의 **식별자 클래스**를 만들어 주어야 한다. 이유는 크게 두 가지다.

1. **조회 메서드의 구조**: 단일 PK(`Long id`)를 쓸 때는 `em.find(Member.class, 1L)`처럼 값 하나만 넘기면 된다. 하지만 PK가 `(student_id, course_id)`처럼 두 개 이상이면, `find()` 메서드에 두 값을 따로 넘길 방법이 없다.
2. **영속성 컨텍스트(1차 캐시)의 `Map` 구조**: JPA의 1차 캐시는 내부적으로 `Map<식별자, 엔티티>` 형태로 데이터를 관리한다. `Map`의 Key는 반드시 **단일 객체**여야 하므로, 여러 개의 PK 컬럼을 자바 세상에서 하나의 객체(식별자 클래스)로 묶어 주어야 한다.

### 식별자 클래스의 4가지 필수 규칙

JPA 스펙상 복합키 식별자 클래스는 다음 4가지 조건을 반드시 만족해야 한다.

1. **기본 생성자 필수**: Hibernate 같은 JPA 구현체가 리플렉션(Reflection)으로 식별자 객체를 동적으로 생성할 때 필요하다.
2. **`public` 클래스**: JPA 프로바이더가 외부 패키지에서도 접근할 수 있어야 한다.
3. **`Serializable` 구현**: 영속성 컨텍스트 내부의 직렬화, 2차 캐시 저장, 분산 환경에서의 객체 전송 등을 위해 반드시 구현해야 한다.
4. **`equals()` & `hashCode()` 오버라이딩**: 1차 캐시(`Map`)에서 키를 비교하고 엔티티의 동일성을 보장하기 위해 필수다.

---

## 2. `@IdClass` vs `@EmbeddedId` 비교

복합키를 매핑하는 두 방식은 **DB 테이블 구조에 맞출 것인가(`@IdClass`)**, <strong>객체지향 설계에 맞출 것인가(`@EmbeddedId`)</strong>의 차이다.

### ① `@IdClass`: 관계형 DB에 가까운 방식

엔티티 안에 PK 컬럼들을 그대로 나열하고, 클래스 상단에 `@IdClass`로 식별자 클래스를 지정한다. 엔티티의 `@Id` 필드명과 식별자 클래스의 필드명이 정확히 일치해야 한다.

```java
// 1. 식별자 클래스 (엔티티의 @Id 필드명과 정확히 일치해야 함)
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
}

// 2. 엔티티
@Entity
@IdClass(EnrollmentId.class)
public class Enrollment {
    @Id
    private Long studentId;
    @Id
    private Long courseId;

    private LocalDateTime enrolledAt;
}
```

### ② `@EmbeddedId`: 객체지향적인 방식

식별자 클래스에 `@Embeddable`을 붙여 하나의 값 타입으로 만들고, 엔티티는 이 식별자 객체를 `@EmbeddedId` 필드 하나로 통째로 가진다.

```java
// 1. 식별자 클래스
@Embeddable
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
}

// 2. 엔티티
@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;

    private LocalDateTime enrolledAt;
}
```

### 한눈에 보는 차이점

| 구분 | `@IdClass` | `@EmbeddedId` |
| :--- | :--- | :--- |
| **설계 관점** | 관계형 DB 친화적 | 객체지향 친화적 (식별자 자체에 메서드 추가 가능) |
| **필드 중복** | 엔티티와 식별자 클래스 양쪽에 필드 중복 선언 | 식별자 클래스에만 선언 (중복 없음) |
| **JPQL 작성** | 간결함 (`WHERE e.studentId = 1`) | 경로가 한 단계 깊어짐 (`WHERE e.id.studentId = 1`) |
| **추천 상황** | 기존 레거시 DB 스키마를 그대로 옮길 때 유리 | 키 개념을 하나의 도메인 객체로 캡슐화할 때 유리 |

---

## 3. `equals()`와 `hashCode()` 오버라이딩이 필수인 이유

이 부분은 **자바 `HashMap`이 Key를 찾는 원리**와 **JPA 1차 캐시**를 함께 떠올리면 쉽게 이해된다.

1. 영속성 컨텍스트의 1차 캐시는 내부적으로 `Map` 구조로 동작한다.
2. `Enrollment`를 저장하면 1차 캐시에 `Key = new EnrollmentId(1L, 2L)` 형태로 보관된다.
3. 이후 같은 트랜잭션 안에서 해당 데이터를 다시 조회하기 위해 아래 코드를 실행한다.

```java
EnrollmentId searchKey = new EnrollmentId(1L, 2L);
Enrollment result = em.find(Enrollment.class, searchKey);
```

4. 1차 캐시(`Map`)는 방금 만든 `searchKey`와 기존에 저장되어 있던 `Key`가 같은지 판별하기 위해 **먼저 `hashCode()`로 같은 버킷인지 찾고, 그다음 `equals()`로 두 객체의 값이 진짜 같은지 최종 확인**한다.

### 오버라이딩을 누락하면 벌어지는 일

`equals()`와 `hashCode()`를 재정의하지 않으면 `Object` 클래스의 기본 구현을 쓰게 되는데, 이는 내부 값이 아니라 <strong>객체의 메모리 주소값(`==`)</strong>을 비교한다.

- 저장할 때 만든 `EnrollmentId(1L, 2L)`과 조회할 때 `new`로 새로 만든 `EnrollmentId(1L, 2L)`은 안에 든 숫자는 같아도 **서로 다른 메모리 주소**를 가진다.
- 결국 1차 캐시는 *"처음 보는 키네?"*라고 판단해 <strong>캐시 미스(Cache Miss)</strong>가 발생하고, 이미 메모리에 올라와 있는데도 불필요한 DB `SELECT` 쿼리가 다시 나가거나 영속성 컨텍스트의 **동일성(Identity) 보장**이 깨지게 된다.

---

## 4. 복합키를 쓸 때 추가로 알아둘 점

### 1) 왜 대리키(`AUTO_INCREMENT`) + `UNIQUE INDEX`를 더 선호할까?

- **외래키(FK) 참조의 복잡성**: 복합키를 가진 테이블을 다른 테이블이 참조하면, 자식 테이블에도 PK 컬럼 2개가 모두 외래키로 따라붙는다. 테이블 관계가 깊어질수록 조인 조건과 매핑 코드가 빠르게 복잡해진다.
- **비즈니스 변경에 취약**: 예를 들어 나중에 *"같은 학생이 같은 강의를 재수강할 수 있다"*로 요구사항이 바뀌면, PK 구조 자체를 뜯어고쳐야 한다. 반면 대리키(`id`)를 PK로 두고 `(student_id, course_id)`에 `UNIQUE` 제약조건을 걸어두면 요구사항이 바뀔 때 제약조건만 유연하게 조정하면 된다.

### 2) 복합키 사용 시 Spring Data JPA `save()` 성능 주의점

복합키는 `@GeneratedValue`를 쓸 수 없어 개발자가 직접 식별자 객체를 만들어 넣어준다.

문제는 Spring Data JPA의 `repository.save(entity)`를 호출할 때다. `SimpleJpaRepository`의 `save()`는 내부적으로 **`id == null`인지 검사해서 새 엔티티(`INSERT`)인지 기존 엔티티(`merge`)인지 판단**한다. 복합키 엔티티는 이미 `id` 객체가 채워져 있으므로 JPA가 **이미 존재하는 데이터라고 오해**해, 굳이 DB에 `SELECT` 쿼리를 한 번 날려본 뒤 데이터가 없을 때 `INSERT`를 실행한다.

따라서 어쩔 수 없이 복합키를 써야 한다면, 엔티티에 `Persistable<ID>` 인터페이스를 구현하고 `@CreatedDate` 필드가 `null`인지 여부로 `isNew()`를 직접 정의해 주어야 불필요한 `SELECT` 쿼리를 막을 수 있다.

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Enrollment implements Persistable<EnrollmentId> {

    @EmbeddedId
    private EnrollmentId id;

    @CreatedDate
    private LocalDateTime createdAt;

    @Override
    public EnrollmentId getId() {
        return id;
    }

    @Override
    public boolean isNew() {
        // createdAt이 아직 null이면 처음 저장되는 새 엔티티로 판단 -> SELECT 없이 바로 INSERT
        return createdAt == null;
    }
}
```
