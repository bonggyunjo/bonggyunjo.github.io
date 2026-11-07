---
title: "CORS의 동작 원리와 브라우저가 Preflight(OPTIONS) 요청을 먼저 보내는 이유"
date: 2026-11-07T19:30:00+09:00
draft: false
summary: "출처(Origin)의 3요소와 SOP(동일 출처 정책)의 목적부터, CORS 에러를 발생시키는 진짜 주체(서버 vs 브라우저), Preflight 사전 요청이 데이터 변조 사고를 막아주는 원리, 그리고 Spring Boot·Nginx·Postman에서의 실무 관점까지 정리합니다."
categories: ["CS"]
tags: ["CORS", "SOP", "Preflight", "HTTP", "Browser", "Security", "Spring Boot", "Nginx", "Web"]
---

## 1. 출처(Origin)의 3요소와 SOP(동일 출처 정책)

CORS를 이해하려면 먼저 웹 브라우저가 말하는 <strong>출처(Origin)</strong>의 기준과 <strong>SOP(Same-Origin Policy, 동일 출처 정책)</strong>가 왜 존재하는지부터 알아야 한다.

### 1) 출처(Origin)를 결정하는 3요소

하나의 URL에서 출처는 **프로토콜(Scheme) + 호스트(도메인) + 포트 번호** 이 세 가지 조합을 의미한다.

```text
https://example.com:443
──────  ─────────── ───
  │          │       └─ 3. 포트 (HTTPS 기본 443)
  │          └───────── 2. 호스트 (도메인)
  └──────────────────── 1. 프로토콜 (HTTPS)
```

기준 URL이 `https://mykot.kr:443`일 때, 세 요소 중 <strong>단 하나라도 다르면 브라우저는 이를 다른 출처(Cross-Origin)</strong>로 간주한다.

- `http://mykot.kr` ➔ 프로토콜이 다름 (**Cross-Origin**)
- `https://api.mykot.kr` ➔ 서브 도메인이 다름 (**Cross-Origin**)
- `http://localhost:5173` (Vue) vs `http://localhost:8080` (Spring Boot) ➔ 포트 번호가 다름 (**Cross-Origin**)

### 2) 브라우저는 왜 SOP(동일 출처 정책)를 기본으로 적용할까?

만약 브라우저에 SOP가 없다면 어떤 일이 벌어질까? 사용자가 로그인해 둔 은행 사이트(`bank.com`)의 세션 쿠키가 브라우저에 남아있는 상태에서, 우연히 해커가 만든 악성 사이트(`evil.com`)에 접속했다고 가정해 보자.

SOP가 없다면 `evil.com` 페이지에 심어진 자바스크립트가 사용자 몰래 `https://bank.com/transfer`로 송금 요청을 보낼 수 있고, 응답 데이터까지 그대로 읽어가 버릴 수 있다. 이를 원천 차단하기 위해 모든 웹 브라우저는 <strong>"현재 접속한 페이지와 출처가 다른 서버로는 자바스크립트로 데이터를 함부로 주고받지 못한다"</strong>는 SOP 보안 규칙을 기본 적용한다.

---

## 2. CORS란 무엇이며, 에러는 누가 발생시키는가?

### 1) 현대 웹 아키텍처와 CORS의 역할

SOP는 강력한 보안 장치이지만, 프론트엔드와 백엔드 서버를 분리해 구축하는 현대 웹 개발 환경에서는 당장 발목을 잡는다.

- **프론트엔드(Vue / React)**: `https://mykot.kr` (로컬 개발 시 `http://localhost:5173`)
- **백엔드(Spring Boot API)**: `https://api.mykot.kr` (로컬 개발 시 `http://localhost:8080`)

두 서버는 명백히 <strong>다른 출처(Cross-Origin)</strong>이지만 정상적인 서비스 운영을 위해 반드시 데이터를 주고받아야 한다. 이때 백엔드 서버가 응답 헤더에 **"이 프론트엔드 출처(`https://mykot.kr`)는 내가 신뢰하는 곳이니 브라우저가 막지 말고 통과시켜 줘"** 라고 명시적으로 허락해 주는 표준 메커니즘이 바로 **CORS(Cross-Origin Resource Sharing, 교차 출처 리소스 공유)** 다.

### 2) 핵심 오해 바로잡기: CORS 에러는 서버가 아니라 브라우저가 발생시킨다

처음 CORS 에러를 마주하면 흔히 *"백엔드 서버가 내 요청을 거부해서 에러가 났구나"* 라고 생각하기 쉽지만, 실제 동작은 정반대다.

1. 브라우저가 백엔드 서버로 API 요청을 보낸다.
2. **백엔드 서버는 요청을 정상적으로 받아 비즈니스 로직을 수행하고 `200 OK` 응답까지 정상 반환한다.**
3. 응답을 돌려받은 **브라우저**가 서버의 응답 헤더를 검사한다.
4. 만약 응답 헤더에 현재 프론트엔드 출처를 허용한다는 `Access-Control-Allow-Origin` 헤더가 없다면, **브라우저가 중간에서 응답 데이터를 폐기해 버리고 콘솔창에 CORS 에러를 띄운다.**

즉, **CORS는 백엔드 서버를 보호하는 기술이 아니라 사용자의 브라우저를 보호하는 브라우저 단의 보안 정책**이다.

---

## 3. Preflight(사전/예비) 요청의 원리와 존재 이유

개발자 도구(Network 탭)를 열어보면 내가 보낸 `DELETE`나 `POST` 요청 직전에 똑같은 URL로 **`OPTIONS` 메서드의 요청**이 먼저 한 번 날아가는 것을 볼 수 있다. 이것이 바로 **Preflight(사전 요청)** 다.

### 1) Preflight 동작 흐름

```text
[브라우저]                                                    [백엔드 서버]
    │                                                              │
    │ ── ① [Preflight: OPTIONS] "나 DELETE 요청과 인증 헤더 보내도 돼?" ──> │
    │ <─ ② [200 OK] "응, 해당 출처의 DELETE와 Authorization 헤더 허용해" ── │
    │                                                              │
    │ ── ③ [실제 요청: DELETE] "/api/posts/10 삭제해 줘" ───────────────> │
    │ <─ ④ [200 OK] "삭제 완료!" ────────────────────────────────────── │
```

브라우저는 실제 요청(본 요청)을 보내기 전에 `OPTIONS` 메서드로 서버에 사전 질의를 보내, 서버가 해당 출처와 HTTP 메서드, 커스텀 헤더를 안전하게 허용하는지 먼저 확인한 뒤에만 본 요청을 전송한다.

### 2) 왜 굳이 번거롭게 Preflight 요청을 먼저 보낼까?

앞서 살펴봤듯 단순 요청에서는 **"서버가 일단 요청을 처리해 응답을 준 뒤, 브라우저가 뒤늦게 응답 헤더를 보고 차단"** 한다. 조회(`GET`) 요청이라면 응답만 버리면 그만이지만, <strong>서버의 DB 상태를 바꾸는 요청(`DELETE`, `PUT`, `POST`)</strong>에서는 치명적인 문제가 생긴다.

#### 만약 Preflight가 없다면 벌어지는 상황

1. 사용자가 악성 사이트(`evil.com`)에 접속한다.
2. 악성 스크립트가 CORS 개념조차 없던 시절에 만들어진 사내 레거시 서버(`company.local/users/delete/1`)로 `DELETE` 요청을 쏜다.
3. 레거시 서버는 요청을 받자마자 **DB에서 1번 유저 데이터를 실제로 영구 삭제**하고 `200 OK`를 응답한다.
4. 브라우저는 응답을 받은 뒤에야 *"어? `Access-Control-Allow-Origin` 헤더가 없네! CORS 위반이다!"* 라며 응답을 버린다.
5. **결과**: 브라우저가 응답은 막았지만, **이미 서버의 DB 데이터는 영구 삭제되어 돌이킬 수 없는 상태**가 된다.

#### Preflight가 이를 방어하는 방식

이 참사를 막기 위해 브라우저는 서버 데이터를 변경할 위험이 있는 요청을 보낼 때, **아무런 부수 효과(Side-effect)도 일으키지 않는 무해한 `OPTIONS` 정찰병(Preflight)을 먼저 파견**한다.

CORS를 모르는 레거시 서버나 해당 출처를 허용하지 않는 서버는 `OPTIONS` 요청에 올바른 CORS 허용 헤더를 돌려주지 못하므로, 브라우저는 **본 요청(`DELETE`) 자체를 아예 출발시키지 않고 차단**한다.

참고로 현대 SPA(Vue, React) 환경에서는 API를 호출할 때 거의 항상 `Content-Type: application/json`이나 `Authorization: Bearer <Token>` 헤더를 포함하므로, 브라우저의 단순 요청(Simple Request) 조건을 벗어나 **사실상 대부분의 REST API 호출이 Preflight(`OPTIONS`) 과정을 거치게 된다.**

---

## 4. 실무 Q&A: 해결 방법과 Postman의 비밀

### Q1. Spring Boot에서 CORS를 해결하는 방법과 Nginx 리버스 프록시로 해결하는 방법의 차이는?

- **Spring Boot 애플리케이션 내부에서 해결하는 방법**:
  1. **컨트롤러/메서드 단위**: 특정 컨트롤러나 메서드 위에 `@CrossOrigin(origins = "...")` 애노테이션을 선언한다.
  2. **전역(Global) 설정**: `WebMvcConfigurer`의 `addCorsMappings()`를 구현하거나, Spring Security를 쓰는 경우 `SecurityFilterChain`에 `CorsConfigurationSource` 빈을 등록해 허용할 출처·메서드·헤더와 Preflight 캐시 시간(`maxAge`)을 일괄 지정한다.
- **Nginx(리버스 프록시) 인프라 레벨에서 해결하는 방법**:
  - 앞단에 Nginx를 두고 `https://mykot.kr`로 들어오는 요청 중 `/api/*` 경로만 내부 백엔드(`localhost:8080`)로 프록시 패스(`proxy_pass`)하도록 묶어주면, <strong>브라우저 입장에서는 프론트엔드와 백엔드가 완전히 동일한 출처(`https://mykot.kr`)</strong>로 보이게 된다.
  - 애초에 Cross-Origin 상황이 사라지므로 CORS 에러나 Preflight 오버헤드 자체가 발생하지 않는다.

### Q2. 왜 Postman이나 `curl`로 호출하면 CORS 에러가 절대 나지 않을까?

실무에서 프론트엔드 개발자가 *"CORS 에러가 나요"* 라고 했을 때, 백엔드 개발자가 Postman으로 API를 호출해 보고 *"포스트맨에서는 200 OK 잘 떨어지는데요? 프론트엔드 코드 문제 아닌가요?"* 라고 답하면 안 되는 이유가 여기에 있다.

- **브라우저**: 사용자가 악성 사이트에 접속했을 때 브라우저에 저장된 쿠키·세션이 도용되는 것을 막아야 하므로 **SOP와 CORS 검증 로직이 탑재**되어 있다.
- **Postman / `curl` / 서버 간 통신(WebClient, RestTemplate)**: 제3자의 악성 스크립트가 실행되는 환경이 아니라 개발자나 서버가 직접 패킷을 쏘는 순수 HTTP 클라이언트이므로 **SOP/CORS 정책 자체가 아예 존재하지 않는다.**

따라서 Postman에서는 서버가 `Access-Control-Allow-Origin` 헤더를 내려주든 말든 원본 응답을 그대로 보여주지만, 실제 브라우저 환경에서 통신이 성공하려면 반드시 **백엔드 서버(또는 앞단 프록시)에서 올바른 CORS 허용 응답 헤더를 내려주도록 설정**해야 한다.
