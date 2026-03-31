---
title: "SQL Injection의 본질과 방어: ORM은 과연 100% 안전할까?"
date: 2026-03-31T15:30:00+09:00
draft: false
summary: "웹 보안의 가장 기초이자 치명적인 취약점인 SQL Injection. 기본적인 동작 원리(OR 1=1)부터 PreparedStatement의 방어 메커니즘, 그리고 최신 ORM 환경에서 발생하는 예외적인 취약점까지 깊이 있게 파헤쳐 봅니다."
categories: ["Security", "Backend"]
tags: ["SQL Injection", "Security", "ORM", "PreparedStatement", "Troubleshooting"]
---

## 1. 가장 오래되었지만, 여전히 치명적인 위협

웹 애플리케이션 보안을 논할 때 절대 빠지지 않는 단골손님이 있습니다. 바로 **SQL Injection(SQL 삽입 공격)** 입니다. 

개발을 처음 배울 때 "쿼리문에 파라미터를 직접 더하지 마라"는 경고를 숱하게 듣지만, 막상 이것이 데이터베이스 레벨에서 어떻게 동작하고 왜 뚫리는지, 그리고 현대의 프레임워크들은 이를 어떻게 방어하고 있는지 본질을 파악하는 것은 매우 중요합니다.

이번 글에서는 SQL Injection의 기초적인 동작 원리를 살펴보고, 현대 ORM 생태계에서 우리가 놓치기 쉬운 보안적 허점들에 대해 정리해 보겠습니다.

---

## 2. SQL Injection, 어떻게 동작하는가?

SQL Injection의 핵심은 **사용자의 입력값이 '데이터'가 아닌 '실행 가능한 코드(쿼리)'로 인식되게 만드는 것** 입니다. 가장 고전적이고 유명한 `OR 1=1` 공격을 통해 이를 확인해 보겠습니다.

### 취약한 코드 예시 (문자열 결합)
만약 로그인 로직을 아래와 같이 문자열 결합(String Concatenation) 방식으로 구현했다고 가정해 봅시다.

```java
String userId = request.getParameter("id");
String userPw = request.getParameter("pw");

// 최악의 쿼리 작성 방식
String query = "SELECT * FROM users WHERE id = '" + userId + "' AND pw = '" + userPw + "'";

```

### 공격 시나리오
공격자가 아이디 입력창에 다음과 같은 악의적인 문자열을 입력합니다.

> **입력값:** `admin' OR 1=1 --`

이 입력값이 쿼리문과 결합되면, 데이터베이스 서버로 전달되는 최종 쿼리는 다음과 같이 변조됩니다.

```sql
SELECT * FROM users WHERE id = 'admin' OR 1=1 --' AND pw = '...'
[쿼리 해석]

id = 'admin' 이거나 (OR)

1=1 (항상 참인 조건)
```

결과적으로 이 쿼리는 **"비밀번호를 묻지도 따지지도 않고, users 테이블의 모든 데이터를 무조건 참(True)으로 반환하라"**는 명령이 됩니다. 해커는 비밀번호를 모르더라도 관리자(admin) 계정으로 로그인이 가능해지는 끔찍한 결과를 초래합니다.

## 3. 프레임워크의 기본 방어: PreparedStatement의 원리
그렇다면 우리는 이를 어떻게 방어하고 있을까요? 핵심 방어책은 바로 PreparedStatement (파라미터 바인딩) 방식입니다.

과거 프레임워크 없이 순수 JDBC만으로 개발할 때는, DB 연결부터 쿼리 문자열 작성, PreparedStatement 선언, 파라미터 세팅, 그리고 ResultSet을 통한 결과 매핑까지 이 모든 과정을 개발자가 일일이 수동으로 구현해야 했습니다. 코드는 길어지고 번거로웠지만, 이 과정을 거쳐야만 안전하게 데이터를 처리할 수 있었죠.

하지만 현대의 프레임워크를 사용하면 이러한 번거로운 과정이 기본적으로 추상화되어 방어됩니다. 프레임워크가 알아서 내부적으로 PreparedStatement를 생성해 주므로, 개발자는 쿼리와 데이터만 넘겨주면 기본적인 SQL Injection 방어가 자동으로 이루어집니다.

<span style="color:red;">주의</span> : 쿼리를 수동으로 작성해야 할 때의 철칙
프레임워크가 많은 것을 도와주지만, 복잡한 통계나 특정 조건 때문에 수동으로 쿼리를 직접 작성해야 하는 상황은 반드시 생깁니다. 이때는 무조건 문자열 결합(+)을 피하고, 환경에 맞춰 ? (JDBC, Spring 등)나 $1 (PostgreSQL, Node.js 등) 와 같은 바인딩 파라미터를 사용해야만 합니다.
안전한 코드 예시

``` Java
String query = "SELECT * FROM users WHERE id = ? AND pw = ?";

// 프레임워크 없이 직접 할 경우의 번거롭지만 안전한 과정
PreparedStatement pstmt = connection.prepareStatement(query);
pstmt.setString(1, userId);
pstmt.setString(2, userPw);
ResultSet rs = pstmt.executeQuery();
```

🔍 왜 ?나 $1을 쓰면 안전할까?

단순히 특수문자를 이스케이프(Escape) 해주기 때문만이 아닙니다. 핵심은 쿼리의 컴파일 시점과 데이터의 삽입 시점을 분리 하는 데 있습니다.
쿼리 구조 미리 분석 (Prepare): DB 엔진은 ?나 $1이 포함된 쿼리 골격 자체를 먼저 분석하고 컴파일하여 실행 계획을 세웁니다.

데이터 바인딩: 이후에 전달되는 사용자 입력값(admin' OR 1=1 --)은 앞서 세워둔 **'구조'** 를 변경할 수 없는 단순한 **'문자열 데이터(Literal)'** 로 취급됩니다.
따라서 DB는 아이디가 문자 그대로 admin' OR 1=1 -- 인 사용자를 찾게 되며, 당연히 매칭되는 데이터가 없으므로 공격은 실패하게 됩니다.

## 4. ORM은 과연 100% 안전할까?
최근 Spring Data JPA(Hibernate), Prisma, TypeORM 같은 ORM 기술이 표준으로 자리 잡으면서, 개발자가 직접 쿼리를 작성할 일은 크게 줄었습니다. 이들 ORM은 내부적으로 PreparedStatement를 완벽하게 지원하므로, 일반적인 CRUD 작업에서는 SQL Injection 공격이 통하지 않습니다.

그렇다면, ORM을 사용하면 SQL Injection 걱정은 아예 안 해도 되는 걸까요?
정답은 "아니오" 입니다. 최근 보안 기사나 기술 블로그를 보면 ORM 환경에서도 SQL Injection 취약점이 발견되는 사례가 종종 보고됩니다.

## ORM 환경에서 발생하는 취약점 케이스
#### 1. 동적 정렬(ORDER BY)에서의 문자열 결합
파라미터 바인딩(?)은 WHERE 절의 값(Value)에는 적용되지만, 테이블명이나 컬럼명에는 적용할 수 없습니다. 만약 사용자가 정렬 기준 컬럼명을 직접 입력하도록 놔둔 채 이를 문자열로 결합한다면 취약점이 발생합니다.

```Java
// JPA 사용 시 취약할 수 있는 예시 (@Query 내부에서의 동적 컬럼)
@Query("SELECT u FROM User u ORDER BY " + userInputColumn)
// 공격자가 userInputColumn에 (CASE WHEN (1=1) THEN id ELSE name END) 와 같은 구문을 주입하면, Blind SQL Injection 공격의 통로가 될 수 있습니다.
```

#### 2. 네이티브 쿼리(Native Query)의 오남용
JPA에서 복잡한 통계 쿼리를 짜기 위해 @Query(nativeQuery = true)를 사용할 때, 바인딩 파라미터(:param)를 쓰지 않고 무심코 + 연산자로 파라미터를 더해버리는 실수가 잦습니다. 이는 고전적인 문자열 결합 공격과 완벽하게 동일한 취약점을 만들어냅니다.

💡 결론 및 회고
도구(ORM)가 발전하면서 보안의 기본적인 부분은 프레임워크가 알아서 처리해 주는 편리한 세상이 되었습니다. 하지만 편리함 뒤에 숨겨진 추상화의 원리를 모르면, 예외적인 상황에서 치명적인 보안 구멍을 만들게 됩니다.  **PreparedStatement** 가 알아서 막아주겠지"라는 맹신보다는, "내가 작성한 코드가 DB 엔진에 어떤 구조로 전달되는가?"를 꿰뚫어 보는 엔지니어링 마인드가 결국 시스템의 견고함을 결정한다는 것을 다시 한번 깨닫습니다.