---
title: "웹 보안 취약점의 공격 타깃과 다층 방어 원리: SQL Injection, XSS 3대 유형, 그리고 보안 헤더"
date: 2026-11-10T21:00:00+09:00
draft: false
summary: "브라우저·백엔드 서버·데이터베이스 각 계층을 노리는 웹 보안 3대장(XSS, CSRF, SQL Injection)의 공격 타깃을 구조적으로 비교하고, PreparedStatement와 ORM의 사각지대, XSS 3가지 공격 유형 및 다층 방어(재귀 Sanitization, HttpOnly, Helmet 보안 헤더)를 종합 정리합니다."
categories: ["CS"]
tags: ["Security", "SQL Injection", "XSS", "CSRF", "PreparedStatement", "ORM", "HttpOnly", "CSP", "Helmet", "Web"]
---

## 1. 웹 보안 3대장의 공격 타깃과 방어선

웹 애플리케이션의 대표적인 보안 취약점인 **XSS**, **CSRF**, **SQL Injection**을 헷갈리지 않고 이해하는 가장 좋은 방법은 <strong>"공격자가 어느 계층(Layer)을 속이려 하는가?"</strong>를 기준으로 바라보는 것이다.

```text
[ 브라우저 (클라이언트) ] ──(HTTP 통신 / 쿠키)──> [ 백엔드 서버 ] ──(SQL 쿼리)──> [ 데이터베이스 ]
          │                                           │                              │
    XSS 공격 타깃                               CSRF 공격 타깃              SQL Injection 공격 타깃
 (브라우저에서 JS 실행)                      (서버를 속여 위조 요청)          (DB 엔진을 속여 쿼리 변조)
```

| 취약점 | 공격 타깃 계층 | 공격 원리 요약 | 핵심 방어선 |
| :--- | :--- | :--- | :--- |
| **XSS** (Cross-Site Scripting) | **브라우저 (클라이언트)** | 피해자의 브라우저에서 공격자가 심어둔 악성 자바스크립트가 실행되게 만듦 | 입력값 무해화(Sanitization), HTML Entity 이스케이프, `HttpOnly` 쿠키, `CSP` 헤더 |
| **CSRF** (Cross-Site Request Forgery) | **백엔드 서버** | 브라우저가 쿠키를 자동 전송하는 특징을 악용해, 로그인된 피해자 몰래 서버로 위조 요청을 보냄 | `SameSite` 쿠키 설정, CSRF 토큰 검증, `Authorization: Bearer` 헤더 인증 |
| **SQL Injection** | **데이터베이스 (DB)** | 사용자 입력값을 단순 데이터가 아닌 실행 가능한 SQL 문법으로 오인하게 만들어 쿼리를 변조함 | `PreparedStatement`(파라미터 바인딩), 동적 컬럼 화이트리스트 검증 |

과거 프로젝트에서 직접 겪고 구현했던 SQL Injection과 XSS 방어 경험을 바탕으로, 각 계층의 방어 원리와 실무 사각지대를 다시 한번 구조화해 정리해 본다.

- **이전 작성 글 참고**:
  - [SQL Injection의 본질과 방어: ORM은 과연 100% 안전할까?](https://bonggyunjo.github.io/troubleshooting/sql-injection-and-orm/)
  - [XSS의 동작 원리와 Express 미들웨어를 활용한 완벽 방어기](https://bonggyunjo.github.io/troubleshooting/xss-and-middle-defense/)

---

## 2. SQL Injection: 데이터베이스(DB) 계층의 방어

### 1) 공격의 본질

SQL Injection의 본질은 **사용자의 입력값을 단순한 '데이터(Literal)'가 아니라 '실행 가능한 쿼리 문법(Code)'으로 오인하게 만드는 것**이다.

```sql
-- 문자열 결합(+)으로 쿼리를 만들었을 때 ' OR 1=1 -- 입력 시 변조 결과
SELECT * FROM users WHERE id = 'admin' OR 1=1 --' AND pw = '...'
```

문자열 결합 방식으로 쿼리를 조립하면 입력값 안에 포함된 작은따옴표(`'`)와 논리 연산자(`OR 1=1`), 주석 기호(`--`)가 기존 SQL 문법 트리에 그대로 개입하여 조건절 전체를 참(`True`)으로 만들어 버린다.

### 2) 핵심 방어 원리: `PreparedStatement`

이를 근본적으로 막아주는 메커니즘이 <strong>`PreparedStatement`(파라미터 바인딩)</strong>이며, 그 핵심은 <strong>[쿼리 골격 컴파일]과 [데이터 바인딩]의 분리</strong>에 있다.

```text
1단계 [쿼리 골격 컴파일]: SELECT * FROM users WHERE id = ? AND pw = ?
   └─> DB 엔진이 ?(플레이스홀더)가 포함된 구문 트리(AST)를 먼저 컴파일하여 실행 계획을 확정함

2단계 [데이터 바인딩]: ? 자리에 "admin' OR 1=1 --" 주입
   └─> 이미 쿼리 구조(문법 트리)가 확정된 상태이므로, 들어오는 값은 어떤 특수문자를 포함하든
       100% '단순 문자열 데이터(Literal)'로만 취급되어 구문 변조가 불가능해짐
```

즉, 단순히 특수문자를 이스케이프 처리하는 수준을 넘어 **데이터가 쿼리 문법 구조를 절대 바꿀 수 없도록 컴파일 시점과 바인딩 시점을 물리적으로 분리**하는 것이 `PreparedStatement`의 방어 원리다.

### 3) ORM의 치명적 사각지대 2가지

Spring Data JPA(Hibernate), Prisma, TypeORM 같은 현대 ORM은 기본적으로 `PreparedStatement`를 사용하므로 일반 CRUD에서는 안전하다. 하지만 다음 두 가지 사각지대에서는 ORM을 쓰더라도 SQL Injection이 그대로 뚫린다.

1. **동적 정렬(`ORDER BY`)과 테이블/컬럼명 동적 지정**:
   - 파라미터 바인딩(`?`)은 `WHERE` 절의 **'값(Value)'** 자리에만 사용할 수 있을 뿐, SQL 구조 자체를 결정하는 **테이블명이나 컬럼명(`ORDER BY column_name`) 자리에는 바인딩이 불가능**하다.
   - 따라서 정렬 기준을 클라이언트로부터 받아 문자열 결합(`"ORDER BY " + sortColumn`)으로 처리하면 `(CASE WHEN (1=1) THEN id ELSE name END)` 같은 구문을 주입하는 **Blind SQL Injection**의 통로가 된다.
   - **해결책**: 동적 정렬 조건은 반드시 백엔드에서 **허용된 컬럼명 목록(Enum 또는 `Set`)만 통과시키는 화이트리스트(Whitelist) 검증**을 거친 뒤 매핑해야 한다.
2. **Native Query의 오남용**:
   - 복잡한 통계 쿼리를 위해 `@Query(nativeQuery = true)`나 Raw Query를 작성할 때, `:param` 바인딩을 쓰지 않고 무심코 `+` 연산자나 문자열 템플릿으로 사용자 입력값을 직접 이어 붙이는 순간 ORM의 보호막은 완전히 무력화된다.

---

## 3. XSS (Cross-Site Scripting): 브라우저 계층의 다층 방어

### 1) 공격의 본질과 3가지 공격 유형

XSS는 웹 애플리케이션에 대한 사용자의 신뢰를 악용하여, **다른 사용자의 브라우저에서 공격자가 주입한 악성 자바스크립트가 실행되게 만드는 공격**이다. 스크립트가 실행되면 세션 쿠키 탈취(`document.cookie`), 가짜 로그인 폼(피싱) 삽입, 키로깅, 악성 사이트 강제 리다이렉트 등이 가능해진다.

악성 스크립트가 피해자의 브라우저에 도달하는 경로에 따라 크게 3가지 유형으로 나뉜다.

| XSS 유형 | 동작 방식 및 특징 | 대표 시나리오 |
| :--- | :--- | :--- |
| **Stored XSS**<br>(저장형 / 영구적) | 공격자가 악성 스크립트를 게시판·댓글·프로필 등에 입력해 **백엔드 DB에 영구 저장**시킴. 이후 해당 페이지를 열람하는 모든 사용자의 브라우저에서 스크립트가 자동 실행됨 | 게시글 본문에 `<script>`를 심어 열람자 전원의 세션을 탈취 (파괴력이 가장 큼) |
| **Reflected XSS**<br>(반사형 / 비영구적) | 악성 스크립트가 포함된 URL 파라미터(예: `?keyword=<script>...`)를 서버가 DB에 저장하지 않고 <strong>응답 페이지 결과에 그대로 포함해 즉시 반사(Reflect)</strong>할 때 발생 | 공격자가 조작된 URL을 피싱 메일이나 메신저로 보내 피해자의 클릭을 유도 |
| **DOM-based XSS**<br>(DOM 기반) | 서버의 응답 HTML 자체는 정상이지만, **브라우저 내부의 클라이언트 자바스크립트가 URL 해시(`location.hash`)나 입력값을 읽어 `innerHTML`, `document.write()` 등으로 DOM을 직접 그릴 때** 발생 | 서버를 거치지 않고 순수하게 브라우저 렌더링 단계에서 발생하므로 서버 로그에 흔적이 남지 않을 수 있음 |

- **참고 자료**: [XSS(크로스사이트 스크립팅)란? 공격 유형 3가지와 대응 방안 정리 (Sparrow)](https://sparrow.im/kr/content-center/xss%ED%81%AC%EB%A1%9C%EC%8A%A4%EC%82%AC%EC%9D%B4%ED%8A%B8-%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8C%85%EB%9E%80-%EA%B3%B5%EA%B2%A9-%EC%9C%A0%ED%98%95-3%EA%B0%80%EC%A7%80%EC%99%80-%EB%8C%80%EC%9D%91/)

### 2) 실전 다층 방어 전략 (Defense in Depth)

XSS는 단일 기법만으로 100% 막기 어렵기 때문에, **입력단 필터링 ➔ 출력단 치환 ➔ 쿠키 접근 제어 ➔ 브라우저 보안 헤더**로 이어지는 다층 방어(Defense in Depth) 체계를 구축해야 한다.

#### ① 입력단 재귀 무해화 (Sanitization) 및 전역 미들웨어 적용

클라이언트가 보내는 JSON 데이터는 단순 1차원 문자열뿐만 아니라 객체 안에 배열과 객체가 중첩된 형태일 수 있다. 따라서 모든 뎁스(Depth)를 끝까지 파고들어 무해화하는 <strong>재귀 함수(`xssClean`)</strong>를 설계하고, 이를 개별 컨트롤러가 아닌 **전역 미들웨어(`app.use`) 또는 글로벌 필터**에 등록해 개발자의 누락(휴먼 에러)을 원천 차단해야 한다.

```javascript
const xss = require('xss');

// 중첩된 객체/배열 JSON까지 끝까지 파고드는 재귀 Sanitization 함수
const xssClean = (obj) => {
  if (typeof obj === 'string') return xss(obj);
  if (typeof obj === 'object' && obj !== null) {
    for (let key in obj) {
      obj[key] = xssClean(obj[key]);
    }
  }
  return obj;
};

// 컨트롤러 진입 전 전역 미들웨어에서 일괄 무해화
app.use((req, res, next) => {
  if (req.body) req.body = xssClean(req.body);
  if (req.query) req.body = xssClean(req.query);
  if (req.params) req.params = xssClean(req.params);
  next();
});
```

#### ② HTML Entity 변환 (이스케이프 처리)

특수문자인 `<`를 `&lt;`로, `>`를 `&gt;`로, `"`를 `&quot;`로 치환(HTML Entity 인코딩)한다. 이렇게 변환된 데이터는 브라우저가 화면에 그릴 때 실행해야 할 HTML 태그가 아니라 **단순 텍스트 문자열**로 인식하므로 스크립트가 실행되지 않는다.

#### ③ `HttpOnly` 쿠키: 세션 하이재킹을 막는 최후의 보루

만약 새로운 우회 패턴으로 인해 화면에서 XSS 스크립트가 실행되더라도, 핵심 자산인 **인증 토큰(세션 ID, Refresh Token)만큼은 절대 털리지 않도록** 쿠키에 `HttpOnly` 플래그를 설정해야 한다. `HttpOnly`가 적용된 쿠키는 브라우저의 자바스크립트 엔진(`document.cookie`)에서 접근하는 것 자체가 물리적으로 차단된다.

#### ④ `Helmet` 등을 활용한 HTTP 보안 응답 헤더 설정

백엔드 서버가 응답을 내려줄 때 보안 헤더를 명시하면 브라우저 차원의 강력한 통제 장치를 활성화할 수 있다.

- **`CSP` (`Content-Security-Policy`)**: 브라우저가 신뢰할 수 있는 출처(`self` 또는 지정된 도메인)의 스크립트만 로드·실행하도록 강제하여, 공격자가 주입한 인라인 스크립트나 외부 악성 스크립트 다운로드를 차단한다.
- **`HSTS` (`Strict-Transport-Security`)**: 브라우저가 해당 도메인에 접근할 때 평문 HTTP 대신 무조건 암호화된 HTTPS 통신만 사용하도록 강제한다.
- **`X-Content-Type-Options: nosniff`**: 공격자가 이미지 파일(`.jpg`)로 위장한 악성 스크립트 파일을 업로드했을 때, 브라우저가 파일 내용을 임의로 추측(MIME Sniffing)해 스크립트로 실행해 버리는 것을 방지한다.

---

## 4. 핵심 요약

1. **공격 타깃의 명확한 구분**: **XSS**는 클라이언트 브라우저에서 악성 JS를 실행시키는 공격이고, **CSRF**는 브라우저의 자동 쿠키 전송을 악용해 백엔드 서버로 위조 요청을 보내는 공격이며, **SQL Injection**은 DB 엔진이 데이터 입력값을 SQL 문법으로 오인하게 만드는 공격이다.
2. **SQL Injection 방어의 핵심**: `PreparedStatement`를 통해 **쿼리 문법 트리의 컴파일 시점과 데이터 바인딩 시점을 분리**해야 한다. 또한 ORM을 사용하더라도 바인딩이 불가능한 `ORDER BY` 동적 컬럼에는 반드시 **화이트리스트(Whitelist) 검증**을 적용해야 한다.
3. **XSS 다층 방어의 핵심**: Stored·Reflected·DOM 기반 XSS를 모두 방어하기 위해 **전역 미들웨어 기반의 재귀 Sanitization과 HTML Entity 치환**으로 1차 방어선을 구축하고, 설령 뚫리더라도 피해를 최소화하도록 **`HttpOnly` 쿠키**와 <strong>HTTP 보안 헤더(`CSP`, `nosniff`, `HSTS`)</strong>로 2중·3중 방어선을 갖춰야 한다.
