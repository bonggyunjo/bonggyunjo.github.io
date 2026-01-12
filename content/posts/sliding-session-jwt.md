---
title: "고정 토큰 만료에 대해 슬라이딩 세션(Sliding Session) 도입기"
date: 2026-01-12T16:50:00+09:00
draft: false
summary: "작업 중 갑작스러운 로그아웃은 사용자에게 치명적인 경험을 선사합니다. HI-REMS 프로젝트에서 Fixed Timeout의 문제를 진단하고, 슬라이딩 세션과 Absolute Timeout을 결합해 UX와 보안을 동시에 잡은 리팩토링 과정을 상세히 기록합니다."
categories: ["Project", "Backend"]
tags: ["JWT", "Node.js", "Express", "Authentication", "UX", "Refactoring", "SlidingSession"]
---

## 1. 시작하며: HI-REMS의 초기 인증 아키텍처

**HI-REMS** 프로젝트를 개발하면서 가장 핵심적으로 설계한 부분 중 하나는 사용자의 보안과 데이터 무결성입니다. 특히 인증 시스템은 서비스의 관문과도 같기에, 보안성이 높은 **Only-Cookie 기반의 JWT(JSON Web Token)** 인증 방식을 채택했습니다.

초기에 구현했던 로그인 흐름은 다음과 같았습니다.
1. 사용자가 이메일과 비밀번호로 로그인을 시도합니다.
2. 서버는 검증 후 유효 기간이 **1시간(60분)**으로 고정된 Access Token을 발급합니다.
3. 클라이언트는 이 토큰을 쿠키에 저장하고 매 요청마다 서버에 전송합니다.
4. 토큰 생성 후 딱 1시간이 지나는 시점에 토큰은 만료되며, 클라이언트의 Axios 인터셉터(Interceptor)가 401 Unauthorized 에러를 감지하여 자동으로 로그아웃 처리를 수행합니다.

이 방식은 구현이 명확하고 보안 정책이 단순하다는 장점이 있었으나, 실제 사용자 환경을 고려했을 때 예상치 못한 변수가 존재했습니다.

---

## 2. 문제의 발견: "열심히 작업 중인데 왜 쫓겨나나요?"

시스템을 직접 테스트하고 운영 시나리오를 점검하던 중, 한 가지 치명적인 UX(사용자 경험) 결함을 발견했습니다. 바로 **'Fixed Timeout(고정 만료)'**에 따른 작업 단절 문제입니다.

HI-REMS는 에너지 데이터를 분석하고 관리하는 플랫폼입니다. 사용자는 복잡한 설정을 변경하거나 혹은 지속적인 모니터링이 필요한 시점이 있는데, 만약 사용자가 로그인한 지 55분이 지난 시점에 아주 중요한 데이터를 입력하거나 모니터링을 하다가 토큰이 만료가 되는 문제에 대해 어떻게 처리할 것인지에 대한 문제를 발견했습니다.

- **데이터 유실 위험**: 사용자가 정성껏 데이터를 입력하고 '저장' 버튼을 누르는 찰나에 1시간이 경과하면, 토큰은 만료됩니다.
- **예상치 못한 인터셉트**: 클라이언트는 401 에러를 받고 즉시 로그인 페이지로 튕겨 나가게 되며, 작성 중이던 데이터는 서버에 도달하지 못한 채 증발해 버립니다.
- **불친절한 흐름**: 사용자가 서비스 내에서 활발히 활동하고 있음에도 불구하고, 시스템은 '최초 로그인 시점'만을 기준으로 사용자를 강제 퇴장시키는 셈입니다.

개발자로서 "기능이 돌아가니까 문제없다"고 넘기기에는 사용자가 겪을 당혹감이 있을거라고 생각했습니다. 이에 활동 중인 사용자에게는 세션을 유연하게 연장해줄 수 있는 대안이 필요했습니다.

---

## 3. 해결책 탐색: 슬라이딩 세션(Sliding Session)도입

이 문제를 해결하기 위해 조사하던 중 **'슬라이딩 세션(Sliding Session)'**이라는 개념을 알게 되었습니다. 마치 자동문의 센서가 사람을 감지할 때마다 문이 열려 있는 시간을 초기화하는 것처럼, 사용자가 API 요청을 보낼 때마다 세션 만료 시간을 자동으로 연장해주는 방식입니다.

하지만 단순히 무한정 연장만 해주는 것은 보안상 위험합니다. 만약 사용자의 PC가 공공장소에서 로그인된 채 방치된다면, 세션이 끊이지 않고 영원히 유지될 수 있기 때문입니다. 

따라서 저는 다음과 같은 **세부 리팩토링 전략**을 세웠습니다.
1. **Sliding Expiration**: 유효한 요청이 들어올 때마다 만료 시간을 갱신한 새 토큰을 발급한다.
2. **Absolute Timeout (절대 만료 시간)**: 사용자가 아무리 활동 중이라도, 최초 로그인 시점으로부터 일정 시간(예: 1시간)이 지나면 보안을 위해 반드시 재인증을 받도록 강제한다.

---

## 4. 리팩토링 구현 상세 (Code Review)

제공해주신 백엔드 코드를 바탕으로 리팩토링된 핵심 로직을 분석해 보겠습니다.

### 4.1 로그인 시점의 '세션 시작점' 기록
먼저 `login` 라우터에서 토큰을 처음 생성할 때, 현재 시각을 `sess`라는 페이로드에 담아 '이 세션의 절대적인 시작점'을 박아둡니다.

```javascript
// auth.router.js
router.post('/login', async (req, res) => {
  // ... 유저 검증 로직 ...
  
  const loginSessionTime = Date.now(); // 세션의 절대적 시작 시간
  
  const access = signAccessToken({ 
    sub: user.member_id, 
    username: user.username, 
    is_admin: user.is_admin 
  }, loginSessionTime); // sess 페이로드에 포함됨

  res.cookie('access_token', access, cookieOpts()).json({ ok: true, token: access });
});
```

### 4.2 인증 미들웨어에서의 슬라이딩 및 절대 만료 검증
- 가장 핵심이 되는 `requireAuth` 미들웨어입니다. 매 요청마다 사용자의 세션 상태를 체크하고 토큰을 갱신합니다.

```javascript
// requireAuth.js
function requireAuth(req, res, next) {
  const bearer = req.headers.authorization || '';
  const token = bearer.startsWith('Bearer ')
    ? bearer.slice(7)
    : (req.cookies && req.cookies.access_token) || '';

  if (!token) return res.status(401).json({ message: 'Unauthorized' });

  try {
    const payload = jwt.verify(token, process.env.JWT_ACCESS_SECRET, {
      algorithms: ['HS256'],
      clockTolerance: 5,
    });

    // 최초 세션 시작 시점 확인
    const sess = typeof payload.sess === 'number'
      ? payload.sess
      : (payload.iat ? payload.iat * 1000 : Date.now());

    const now = Date.now();
    const ABSOLUTE_MAX_MS = getExpiresInMs(); 
    
    // Absolute Timeout 검증
    // 활동 여부와 관계없이 최초 로그인 후 설정된 시간이 지났다면 강제 로그아웃
    if (now - sess > ABSOLUTE_MAX_MS) {
      return res.status(401).json({ message: 'Session expired (Absolute timeout)' });
    }

    // Sliding Session - 새 토큰 발급
    // 검증을 통과했다면 기존 sess(시작점)는 유지한 채, 만료 시간만 갱신된 새 토큰을 생성
    const newAccess = signAccessToken(
      { sub: payload.sub, username: payload.username, is_admin: payload.is_admin },
      sess
    );

    // 클라이언트에 갱신된 토큰 전달
    res.setHeader('X-New-Token', newAccess);
    res.setHeader('Access-Control-Expose-Headers', 'X-New-Token');

    if (res.cookie) {
      res.cookie('access_token', newAccess, cookieOpts());
    }

    req.user = { 
      sub: payload.sub, 
      username: payload.username, 
      is_admin: !!payload.is_admin 
    };
    return next();
  } catch (e) {
    return res.status(401).json({ message: 'Invalid or expired token' });
  }
}
```

## 5. 결과 및 기대 효과

> 이번 리팩토링을 통해 `슬라이딩 세션`이란 무엇인지 알게 되었으며, 프로젝트에서 기술적 개선을 이루었습니다.

무한 연장을 허용하지 않는 `Absolute Timeout`을 통해, 세션 탈취 리스크를 관리하면서도 사용자 편의성을 극대화 하였으며, 중요한 데이터를 입력하거나 실시간 모니터링을 수행하는 도중에 세션이 만료되어 작업 내용이 날아가는 불상사를 차단했습니다.


## 6. 마치며..

"처음에는 단순히 '기능이 동작하는 것'에만 집중했습니다. 하지만 개발자가 아닌 사용자의 입장에서 깊게 고민해 보니, 고정된 1시간이라는 벽은 시스템의 편의일 뿐 사용자에게는 보이지 않는 장애물이었습니다. 현업의 방식을 탐구하고 인증 메커니즘의 본질을 파고들며 도입한 '슬라이딩 세션'은, 기술적 해결을 넘어 사용자 경험을 최우선으로 생각하는 개발자로 한 단계 성장하는 소중한 계기가 되었습니다."