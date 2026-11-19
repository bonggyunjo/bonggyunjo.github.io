---
title: "JPA 엔티티 4가지 상태(비영속, 영속, 준영속, 삭제)와 Detached 상태에서 변경 감지 실패 원인 및 해결"
date: 2026-11-19T15:30:00+09:00
draft: false
summary: "영속성 컨텍스트의 4가지 핵심 기능과 flush(), 벌크 연산으로 인한 데이터 불일치 및 clearAutomatically의 부작용, 그리고 JPA 엔티티 4가지 상태(비영속·영속·준영속·삭제)와 준영속(Detached) 상태에서 변경 감지가 실패하는 원인과 해결 방법을 정리합니다."
categories: ["Backend"]
tags: ["JPA", "Hibernate", "Persistence Context", "Dirty Checking", "Detached", "Spring Data JPA", "Backend"]
---

## 1. 영속성 컨텍스트(Persistence Context)와 핵심 기능

JPA에서 엔티티의 상태를 이해하려면 먼저 <strong>영속성 컨텍스트(Persistence Context)</strong>가 무엇인지부터 짚어야 한다.

**영속성 컨텍스트**는 엔티티를 영구 저장하는 환경이자, 애플리케이션과 데이터베이스 사이에서 객체를 보관·관리하는 논리적인 가상 공간이다. 단순히 객체를 담아두는 임시 저장소가 아니라, 데이터 일관성과 성능을 위해 다음 4가지 핵심 기능을 제공한다.

1. **1차 캐시**: `em.find()`로 데이터를 조회할 때 DB를 먼저 뒤지지 않고 1차 캐시부터 확인한다. 캐시에 해당 엔티티가 있으면 DB 조회 없이 메모리의 객체를 바로 반환해 성능을 높인다.
2. **동일성(Identity) 보장**: 같은 트랜잭션 안에서 같은 식별자(PK)로 엔티티를 두 번 조회하면 1차 캐시에 보관된 동일한 객체 인스턴스를 반환한다. 즉, 자바 컬렉션에서 꺼낸 객체처럼 `a == b` 비교가 `true`가 된다.
3. **트랜잭션을 지원하는 쓰기 지연 (Transactional Write-Behind)**: `em.persist()`를 호출해도 곧바로 `INSERT` SQL을 DB로 보내지 않고, 내부의 **쓰기 지연 SQL 저장소**에 쿼리를 모아둔 뒤 한꺼번에 전송해 네트워크 왕복을 줄인다.
4. **지연 로딩 (Lazy Loading)**: 연관된 엔티티를 처음부터 조인(Join)해 한꺼번에 조회하지 않고, 실제 그 데이터가 사용되는 시점에 쿼리를 날려 불필요한 조회 비용과 메모리 낭비를 줄인다.

### 플러시(`flush()`)란?

영속성 컨텍스트에 쌓인 변경 내용을 실제 DB에 동기화하는 작업을 <strong>플러시(Flush)</strong>라고 한다. 쓰기 지연 SQL 저장소에 모여 있던 등록·수정·삭제 쿼리는 **트랜잭션이 커밋되거나 JPQL이 실행될 때** 플러시가 발생하면서 실제 DB에 반영된다.

---

## 2. 벌크 연산과 영속성 컨텍스트 초기화(`clear`)의 부작용

과거 트러블슈팅 글([JPA 연관 데이터 삭제 버그, flush()와 clear() 적용 (영속성 컨텍스트)](https://bonggyunjo.github.io/troubleshooting/persistencecontext/))에서도 다룬 적 있지만, 영속성 컨텍스트 개념에서 반드시 함께 짚고 넘어가야 할 부분이 <strong>벌크 연산(Bulk Operation)</strong>이다.

벌크 연산은 수많은 레코드를 한 번의 쿼리로 수정/삭제하므로 속도가 매우 빠르지만, **영속성 컨텍스트(1차 캐시)를 완전히 무시하고 곧바로 DB에 직접 쿼리를 실행한다**는 무서운 부작용이 있다.

```text
1. 메모리에 '홍길동(나이 20살)'을 올려둠 (1차 캐시에 영속 상태로 존재)
2. 벌크 연산 실행: "모든 사람 나이 1살씩 올려라" (1차 캐시를 거치지 않고 DB로 직행)
   ➔ DB에는 홍길동 나이가 '21살'로 바뀜
3. 그런데 내 1차 캐시 안의 홍길동은?
   ➔ 여전히 '20살'로 남아 있음! (DB와 메모리 간 데이터 불일치 발생)
```

이 데이터 불일치를 막기 위해 Spring Data JPA에서는 벌크 연산 메서드에 `@Modifying(clearAutomatically = true)` 옵션을 제공한다.

하지만 당시 겪었던 버그처럼, 회원 탈퇴 도중 호출한 연관 데이터 정리 메서드 안에 `@Modifying(clearAutomatically = true)` 벌크 연산이 들어 있으면 **트랜잭션 중간에 영속성 컨텍스트가 통째로 비워진다(clear).** 그 바람에 이미 영속 상태였던 `user` 엔티티가 졸지에 **준영속(Detached)** 상태로 밀려나면서, 뒤이어 호출한 변경 감지(`withdraw()`)와 연관관계 삭제가 꼬여버리는 문제가 발생할 수 있다.

---

## 3. JPA 엔티티의 4가지 상태

엔티티 객체는 영속성 컨텍스트와의 관계에 따라 다음 4가지 생명주기 상태를 가진다.

```text
           ┌─────────────── (em.persist) ───────────────┐
           ▼                                            │
[비영속 (New)] ➔ [영속 (Managed)] ──(em.detach / clear / 트랜잭션 종료)──> [준영속 (Detached)]
                       │
                       └─────────────── (em.remove) ──────────────> [삭제 (Removed)]
```

### 1) 비영속 (New / Transient)

- 자바 메모리에서 `new` 키워드로 막 생성된 **순수 자바 객체** 상태로, 영속성 컨텍스트나 DB와 전혀 관련이 없다.
- 아직 영속성 컨텍스트가 관리하지 않으므로 1차 캐시나 변경 감지 기능이 동작하지 않는다.

```java
Item item = new Item(); // 비영속 상태
item.setName("노트북");
```

### 2) 영속 (Managed)

- 영속성 컨텍스트에 저장되어 JPA의 관리를 받는 상태다.
- `em.persist(entity)`를 호출하거나, `em.find()` / Spring Data JPA `findById()` / JPQL 등으로 조회한 엔티티가 여기에 해당한다.
- 1차 캐시에 저장되며, 엔티티의 필드 값을 바꾸면 트랜잭션 커밋(플러시) 시점에 <strong>변경 감지(Dirty Checking)</strong>를 통해 `UPDATE` 쿼리가 자동 실행된다.

```java
Item item = itemRepository.findById(1L).get(); // 영속 상태
item.setPrice(50000); // 별도 save() 호출 없이도 커밋 시 UPDATE 실행
```

### 3) 준영속 (Detached)

- 영속성 컨텍스트에서 **분리(`detach`, `clear`, `close`)되었거나 트랜잭션이 끝나서 더 이상 JPA의 감시를 받지 않는 상태**다.
- 겉보기에는 DB 데이터와 똑같은 값을 들고 있어 헷갈리기 쉽지만, 값을 아무리 바꿔도 JPA가 추적하지 않으므로 **변경 감지(`UPDATE`)가 절대 일어나지 않는다.**

### 4) 삭제 (Removed)

- 영속성 컨텍스트와 DB에서 삭제하기로 예정된 상태다.
- 영속 상태의 엔티티에 `em.remove(entity)`를 호출하면 삭제 쿼리가 쓰기 지연 SQL 저장소에 등록되고, 플러시 또는 트랜잭션 커밋 시점에 실제 DB에서 `DELETE` 쿼리가 실행된다.

---

## 4. Detached(준영속) 상태에서 변경 감지가 실패하는 원인과 해결

### 왜 변경 감지가 실패할까?

준영속 상태에서 변경 감지가 실패하는 핵심 원인은 **영속성 컨텍스트가 해당 엔티티를 관리 및 스냅샷 비교 대상에서 제외했기 때문**이다.

1. JPA의 <strong>변경 감지(Dirty Checking)</strong>는 엔티티를 처음 1차 캐시에 올릴 때 찍어둔 **최초 스냅샷**과 트랜잭션 종료(플러시) 시점의 **현재 엔티티 상태**를 비교해 바뀐 컬럼이 있을 때 `UPDATE` 쿼리를 만든다.
2. 하지만 `em.detach()`, `em.clear()`, `em.close()`가 호출되거나 `@Transactional` 범위가 종료되면, 엔티티는 영속성 컨텍스트의 **1차 캐시와 스냅샷 저장소에서 제거**된다.
3. 결국 데이터베이스와의 연결 고리가 끊어진 <strong>JVM 메모리의 평범한 자바 객체(POJO)</strong>로 돌아가기 때문에, 아무리 필드를 수정해도 비교할 스냅샷 기준 자체가 없어 `UPDATE` 쿼리가 전혀 생성되지 않는다.

### 해결 방법

준영속 상태에서 변경이 누락되는 문제를 막으려면 엔티티가 <strong>영속 상태(Managed)</strong>일 때 수정이 일어나도록 흐름을 잡아야 한다.

#### 1) 트랜잭션 경계(`@Transactional`) 안에서 조회 및 수정

스프링 부트 환경에서는 기본적으로 트랜잭션의 생명주기와 영속성 컨텍스트의 생명주기가 일치한다.

- **문제 발생 원인**: 트랜잭션이 없는 컨트롤러(Controller) 계층이나 `@Transactional`이 끝난 이후에 엔티티의 필드 값을 수정하면, 이미 영속성 컨텍스트가 종료되어 엔티티가 준영속 상태가 된다. 이때는 값을 아무리 바꿔도 DB에 반영되지 않는다.
- **해결 원리 및 동작 흐름**: `@Transactional`이 붙은 서비스(Service) 메서드 내부에서 엔티티를 조회하고 값을 변경하면, 메서드가 시작될 때 열린 영속성 컨텍스트가 메서드가 끝날 때까지 유지된다. 이후 메서드 종료 시점에 트랜잭션 커밋과 플러시가 발생하며 최초 스냅샷과 비교해 변경 감지가 정상적으로 실행된다.

```java
// 올바른 흐름: 서비스 계층 내부에서 조회와 수정을 한 번에 처리
@Service
public class MemberService {

    @Transactional
    public void updateName(Long id, String newName) {
        // 1. 트랜잭션 시작 -> 영속성 컨텍스트 생성 및 영속 상태 엔티티 조회
        Member member = memberRepository.findById(id).orElseThrow();

        // 2. 영속 상태에서 값 변경
        member.changeName(newName);

        // 3. 메서드 종료 시 커밋 -> 변경 감지 동작 (UPDATE 쿼리 실행)
    }
}
```

#### 2) 벌크 연산(`clearAutomatically = true`) 이후 엔티티 재조회

JPQL이나 `@Modifying`을 사용한 벌크 연산(다건 일괄 수정/삭제)은 영속성 컨텍스트를 거치지 않고 DB에 직접 SQL을 실행한다.

- **문제 발생 원인**: DB 데이터는 이미 바뀌었지만 영속성 컨텍스트에는 과거 데이터가 남아 데이터 불일치가 발생한다. 이를 막기 위해 `@Modifying(clearAutomatically = true)` 옵션을 주면 벌크 연산 직후 영속성 컨텍스트를 강제로 비우게(`em.clear()`) 되는데, 이때 기존에 자바 변수로 들고 있던 엔티티는 순간적으로 준영속 상태가 된다.
- **해결 원리 및 동작 흐름**: 영속성 컨텍스트가 초기화된 이후에는 기존 변수를 재사용하지 말고 `findById()` 등으로 DB에서 최신 데이터를 다시 조회해야 한다. 재조회한 엔티티는 영속성 컨텍스트의 1차 캐시에 새로 등록되어 다시 변경 감지의 대상이 된다.

```java
@Transactional
public void bulkAndSingleUpdate(Long memberId) {
    Member member = memberRepository.findById(memberId).orElseThrow();

    // 1. 벌크 연산 실행 -> clearAutomatically = true 옵션으로 영속성 컨텍스트 초기화
    // 이 순간 위에서 조회한 'member' 변수는 준영속 상태가 됨
    memberRepository.bulkAgePlus(20);

    // [위험] 준영속 상태가 된 기존 변수를 수정하면 DB에 반영되지 않음
    // member.changeName("변경실패");

    // [해결] 영속성 컨텍스트가 비워졌으므로 DB에서 다시 조회하여 영속 엔티티 획득
    Member refreshedMember = memberRepository.findById(memberId).orElseThrow();
    refreshedMember.changeName("변경성공"); // 정상 반영
}
```

#### 3) `merge()` 대신 변경 감지(재조회) 사용

준영속 상태의 엔티티를 다시 영속 상태로 되돌리는 방법은 크게 `em.merge()` 호출과 재조회 후 변경 감지 두 가지가 있다.

- **`merge()`의 위험성**: `em.merge(detachedEntity)`를 호출하면 JPA는 해당 엔티티의 식별자(PK)로 DB에서 데이터를 조회한 뒤, 파라미터로 넘어온 준영속 엔티티의 **모든 필드 값**을 기존 엔티티에 통째로 덮어씌운다. 만약 클라이언트에서 수정하지 않아 `null`로 넘어온 필드가 있다면, DB에 잘 저장되어 있던 기존 데이터까지 `null`로 덮어쓰는 데이터 유실 사고가 발생할 수 있다.
- **재조회 후 변경 감지의 장점**: 트랜잭션 안에서 `findById()`로 DB의 원본 엔티티를 먼저 영속 상태로 불러온 뒤, 수정이 필요한 특정 필드만 비즈니스 메서드로 골라서 변경하면 원본 데이터가 보존된 상태에서 필요한 필드만 안전하게 업데이트된다.

```java
// 안전한 패턴: DTO로 필요한 값만 받아 영속 엔티티의 특정 필드만 선택적으로 수정
@Transactional
public void updateMemberSafe(Long memberId, MemberUpdateDto dto) {
    // 1. DB에서 원본 엔티티 조회 (기존 데이터 모두 안전하게 유지됨)
    Member member = memberRepository.findById(memberId).orElseThrow();

    // 2. DTO에 담긴 변경할 필드만 선택적으로 수정 (다른 필드는 건드리지 않음)
    if (dto.getName() != null) {
        member.changeName(dto.getName());
    }
}
```
