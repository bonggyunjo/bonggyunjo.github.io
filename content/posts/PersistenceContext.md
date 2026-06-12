---
title: "JPA 연관 데이터 삭제 버그, flush()와 claer() 적용(영속성 컨텍스트)"
date: 2026-06-12T10:30:00+09:00
draft: false
summary: "영속성 컨텍스트와 구현을 하면서 마주친 문제점"
categories: ["Project", "Backend"]
tags: ["영속성 컨텍스트", "JPA", "flush()", "clear()", "Cascade", "Casecade.ALL", "persist", "Persist", "Modifying"]
---

## 들어가며: 문제의 발단

플램폼 내 회원 탈퇴 기능 중, 발생한 문제가 있었다. 

기능 중 회원 탈퇴 시 유저 데이터를 DB에서 완전히 날려버리는 물리적 삭제(Hard Delete) 대신, 복구 및 운영 목적을 위해 deleted = true 형태로 상태만 변경하는 논리적 삭제(Soft Delete) 방식을 채택하고 있다. 반면, 해당 유저가 작성한 게시글, 좋아요, 그리고 유저와 강하게 결합되어 있는 전문가(Supporter) 프로필 등의 연관 데이터는 깔끔하게 물리적 삭제 및 익명화 처리를 해야 했다.

그런데 테스트를 해보니 유저 본체의 Soft Delete는 정상적으로 이루어지는데, User와 1:1로 강하게 결합된 Supporter 테이블의 데이터는 삭제되지 않는 문제가 발생했다. 심지어 이 남아있는 좀비 데이터 때문에 이후 로직에서 EntityNotFoundException이 터지며 트랜잭션이 롤백되는 현상까지 나타났다.

여기서 문제는 Modifying과 영속성 컨텍스트간 불일치 문제였다.

---

## 도메인 구조와 기존 탈퇴 로직

문제를 파악하기 위해 먼저 User와 Supporter의 연관관계 매핑 상태를 확인했다.

```java
//User.java(Entity)
@OneToOne(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
private Supporter supporter;
```

위와 같이 User 엔티티는 CascadeType.ALL과 orphanRemoval = true가 걸려있었다. CascadeType.ALL은 CRUD 연산을 모두 전이한다는건데 즉, 부모인 User의 상태 변화가 자식인 Supporter에게 모두 전이되어야 정상이다.

기존 탈퇴 로직의 대략 흐름은 다음과 같았다.

```java
@Transactional
public void deleteUserByUser(Long userId, UserWithdrawRequest request) {
    // 1. 유저 및 서포터 조회 (영속 상태)
    User user = userRepository.findById(userId).orElseThrow(...);
    
    // 2. 서포터 연관관계 끊기 및 삭제
    supporterRepository.findByUserId(userId).ifPresent(supporter -> {
        user.setSupporter(null);
        supporterRepository.delete(supporter);
    });

    // 3. 연관 데이터 일괄 삭제 및 익명화 (게시글, 좋아요 등)
    userCleanupService.cleanupUserData(userId);
    
    // 4. 유저 상태 Soft Delete
    user.withdraw(); // 이대로 트랜잭션을 종료시켰다.
}
```

그리고 cleanupUserData는 UserCleanupService에서 연관된 기타 데이터(좋아요,게시글 등)을 일괄 삭제 및 익명화하는 메서드를 만들고, UserService에서 호출해 유저와 연관된 기타 데이터를 일괄 삭제 및 익명화한다.

```java
// UserCleanupService.java 중 일부

public void cleanupUserData(Long userId) {
    // ... (중략) ...
    routeLikeRepository.anonymizeUserLikes(userId);
    postLikeRepository.anonymizeUserLikes(userId);
    postRepository.anonymizeRouteReviews(route.getId());
    // ...
}
```

이떄 문제는 3번의 userCleanupService. 내부에 있는 벌크 연산 코드에서 시작되고 있었다.
★★★ 문제는 Modifying과 영속성 컨텍스트간 불일치 문제였다. ★★★

```java
//UserService.java 
userCleanupService.cleanupUserData(userId);
```

anonymizeUserLikes, anonymizeUserLikes, anonymizeRouteReviews등 각 도메인 계층의 repository에서 성능 최적화를 위해 데이터를 지우거나 익명화하도록 JPA의 변경 감지 대신 @Modifying 어노테이션을 활용한 벌크 연산 쿼리로 구현되어 있었다.

```java
// RouteLikeRepository.java 중 일부
    @Modifying(clearAutomatically = true)
    @Query("update RouteLike rl set rl.user = null where rl.user.id = :userId")
    void anonymizeUserLikes(@Param("userId") Long userId);
```

★★★ 이때 "벌크 연산"이 바로 문제였다. ★★★

## 영속성 컨텍스트가 초기화되며 발생한 "준영속" 
JPA를 쓸 때 성능 최적화를 위해 @Modifying을 사용한 벌크 연산을 자주 사용한다. 여기서 @Modifying은 영속성 컨텍스트를 거치지 않고 직접 데이터베이스(DB)로 쿼리를 전달하는데, 
결론부터 말하면 이때 벌크 연산 직후와 직전에 조회한 데이터 간 불일치 즉, 영속성 컨텍스트엔 캐시가 남아있어 문제가 발생하고 있었다. 
벌크 연산은 영속성 컨텍스트(1차 캐시)를 무시하고 DB에 직접 쿼리를 날리기 때문에, DB와 1차 캐시 간의 상태 불일치가 발생할 위험이 있다.

이를 방지하기 위해 관행적으로 clearAutomatically = true 옵션을 주어 벌크 연산 직후 영속성 컨텍스트를 깔끔하게 비워버린다.

트랜잭션 시작: findById()로 User를 불러온다. 이때 User는 1차 캐시에서 관리되는 '영속(Managed)' 상태다.

삭제 대기: user.setSupporter(null) 및 delete(supporter)를 호출한다. 아직 DB에는 안 갔고, 쓰기 지연 저장소에 모여있다.

벌크 연산 실행: userCleanupService가 호출되며 @Modifying(clearAutomatically = true)가 실행된다.

1차 캐시 폭파: 쿼리가 DB로 날아간 직후, 영속성 컨텍스트가 통째로 초기화(clear) 되어버린다!

준영속 상태로 전락: 영속성 컨텍스트가 초기화되었으므로, 1번 단계에서 조회해 우리가 열심히 조작하고 있던 user 객체는 졸지에 JPA의 관리 대상에서 벗어난 '준영속(Detached)' 상태가 되어버린다.

변경 감지(Dirty Checking) 실패: 마지막에 user.withdraw()를 호출해 값을 바꿨지만, user는 이미 준영속 상태이므로 트랜잭션이 끝날 때 JPA가 변경된 값을 DB에 반영해 주지 않는다. 게다가 CascadeType.ALL 설정 때문에 꼬여버린 객체 참조가 에러(EntityNotFoundException)를 유발한 것이다.

때문에, users는 soft_delete가 되었지만 이를 참조하고있는 supporter는 영속성 컨텍스트로 인해 삭제가 되지 않았고 이러한 문제점들이 발생한것이다.

## 해결책
결국 DB에서 가장 최신의 데이터를 불러와서 영속 상태로 만든 후, 마무리를 짓도록 하기 위해 cleanupUserData() 수행 후, 캐시(영속성 컨텍스트) 불일치를 해결하기 위해 flush()와 clear()를 호출하였다.

```java
userCleanupService.cleanupUserData(userId);

// 초기화
userRepository.flush();
entityManager.clear();

//  유저를 새로 조회 (영속 상태로)
User cleanUser = userRepository.findById(userId).orElseThrow(...);

// Soft Delete 수행
cleanUser.withdraw();

// 저장
userRepository.save(cleanUser);
```
