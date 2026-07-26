---
title: "Next.js 미들웨어 — 요청을 가로채는 검문소 (feat. Spring Filter/Interceptor)"
date: 2026-07-27 01:20:47 +0900
categories: [공부, Next.js]
tags: [nextjs, 미들웨어, middleware, 인증, spring]
---

## 오늘 공부한 것
미들웨어를 읽었다. 원래 CORS 때문에 이름만 알고 있었는데, 읽다 보니 쿠키·path rewriting·matcher 등 훨씬 많은 걸 한다길래 정리. Spring 배경이 있어서 Filter/Interceptor에 매핑하며 이해했다.

## 내가 가졌던 질문
- 미들웨어가 CORS 말고 뭘 하나
- 왜 `NextRequest`/`NextResponse`를 쓰나, 그냥 `Request`/`Response`는 안 되나
- Next 안 쓰고 순수 React만 쓰면 미들웨어는?
- 미들웨어가 API 통신의 "다리" 역할인가
- 검문소로서 실제로 하는 일이 뭔가
- 로그인 안 하면 `/login`으로 튕기는 그 역할 맞나
- Spring에선 이걸 어디서 하나 (service? controller?)
- Spring Boot도 미들웨어를 만드나
- 미들웨어 개념 자체가 Next에서 나온 건가

## 정리 (Q&A 핵심)

### 미들웨어 = 다리가 아니라 검문소
처음엔 CORS 기억 때문에 "프론트·백엔드를 **연결**해주는 다리"로 생각했는데, 정확히는 **검문소(톨게이트)**다.

```
다리(bridge)   : 양쪽을 이어줌.        목적 = 연결
검문소(톨게이트) : 지나가는 걸 세워 검사. 목적 = 통제·판단
```

미들웨어는 요청이 목적지(page/route)로 가는 **길목에 세운 검문소**다. 요청은 원래도 목적지로 가던 중이었고, 미들웨어는 그 길을 **가로채서 검사**할 뿐. 뭔가를 연결·중개하는 게 아니다. CORS(응답 헤더 하나 붙이기)는 검문소가 하는 여러 일 중 곁다리고, 주력은 인증 가드다.

- API 통신 전용도 아니다. `matcher`로 지정한 **모든 요청**(화면 page 요청 포함)에 세울 수 있다.

### 파일 하나 — `middleware.ts`
프로젝트 루트에 딱 하나(`app/`과 같은 층). 이름 고정.

```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  return NextResponse.next();  // "통과시켜라"
}
```

### 검문소가 하는 일 — 읽고 → 판단하고 → 보낸다
```
① 읽는다   : 쿠키·헤더·URL·지역 등 판단 재료 수집
② 판단한다 : 통과? 차단? 우회?
③ 보낸다   : next(통과) / redirect(돌려보냄) / rewrite(우회)
```

| 반환 | 뜻 | 예 |
|---|---|---|
| `NextResponse.next()` | 통과 | 로그인됨 → 목적지로 |
| `NextResponse.redirect(url)` | 돌려보냄 | 로그인 안 됨 → `/login` |
| `NextResponse.rewrite(url)` | URL 그대로, 내용만 딴 길 | 지역/언어 분기, A/B |

실전 임무: **인증 가드(압도적으로 흔함)**, 쿠키 굽기(`res.cookies.set`), 헤더 손질, 지역/언어 분기, 출입 통제(점검중·IP 차단).

단 Edge에서 돌기 때문에 **가벼운 판단만** 한다. "토큰 있나 없나"까지가 미들웨어 몫이고, "이 토큰이 진짜 유효한지 DB 조회" 같은 무거운 검증은 목적지(page/route)에서.

### redirect vs rewrite
| | redirect | rewrite (path rewriting) |
|---|---|---|
| URL 바뀌나 | ✅ 바뀜 | ❌ 그대로 |
| 사용자가 아나 | 봄 (주소창 바뀜) | 모름 (몰래 딴 내용) |
| 예 | 로그인 안 됨 → `/login` | `/ko/about`인데 속으론 `/about` |

- redirect = "너 저기로 가" (다른 문으로 보냄)
- rewrite = "주소는 그대로, 안에서 몰래 다른 방 내용을 줌"

### 왜 `NextRequest`/`NextResponse`인가
결론: **미들웨어가 하는 일 자체가 표준엔 없고 Next 버전에만 있는 기능**이라서.

- `NextRequest`/`NextResponse`는 표준 `Request`/`Response`를 **상속한 확장판**(같은 것 + 추가 기능).
- `rewrite()`, `next()`는 표준 `Response`에 **아예 없다** → 표준으론 미들웨어를 못 짠다.
- 쿠키도 표준은 헤더 문자열을 직접 파싱해야 하지만, `NextRequest`는 `request.cookies.get('token')` 한 줄.
- 대조: API 라우트(`route.ts`)는 body만 만지는데 `json()`은 표준·Next 둘 다 똑같아서 표준으로도 충분했다. 미들웨어는 쿠키·redirect·rewrite가 일이라 Next 버전이 필수.

또 request 쪽은 애초에 Next가 `NextRequest`를 쥐여준다(내가 고르는 게 아님).

### 순수 React엔 미들웨어가 없다
미들웨어는 "Next 전용 기능을 쓰는 것"이 아니라, **미들웨어 자체가 서버 프레임워크 기능**이다.

```
순수 React = 브라우저에서 도는 CSR → 요청을 가로챌 서버가 없음 → 미들웨어 개념 자체가 없음
Next.js   = React + 서버 레이어    → 그 서버 앞에서 가로챌 수 있음 → 미들웨어 존재
```

순수 React로 로그인 가드를 하려면 클라이언트에서 `useEffect`로 체크하거나(이미 화면 받은 뒤라 덜 안전), react-router 가드, 또는 **별도 백엔드 서버**(Express 등)를 붙여 그 서버의 미들웨어를 써야 한다. "미들웨어"는 항상 **서버 프레임워크에 딸린 물건**이다.

### Spring 매핑 — 미들웨어의 짝은 service가 아니라 Interceptor/Filter
"Spring에선 service에서 if로 팅겨내면 되지 않나?" 싶었는데, 층이 하나 어긋난다. service는 이미 목적지(controller) 안이고, 미들웨어의 정확한 짝은 **controller 도착 전에 가로채는 `Filter`/`Interceptor`**다.

`HandlerInterceptor.preHandle`이 미들웨어와 1:1이다.

```java
public boolean preHandle(request, response, handler) {
    if (쿠키에 토큰 없음) {
        response.sendRedirect("/login");
        return false;   // controller 안 감 (튕김)
    }
    return true;        // 통과
}
```

| Spring `preHandle` | Next 미들웨어 |
|---|---|
| `return false` + `sendRedirect` | `NextResponse.redirect()` |
| `return true` | `NextResponse.next()` |
| `addPathPatterns("/dashboard/**")` | `matcher: ['/dashboard/:path*']` |

controller의 `@ResponseBody`는 "튕김"이 아니라 **검문 통과한 요청에 주는 성공 결과**다. 무거운 검증은 service에서.

| 역할 | Next.js | Spring |
|---|---|---|
| 검문소(인증 가드) | `middleware.ts` | Filter / Interceptor / Security |
| 목적지 엔드포인트 | `route.ts`/`page.tsx` | Controller |
| 비즈니스 로직 | route 함수 안 | Service |

### 인증 가드가 핵심 — 왜 미들웨어에서 하나
보호된 페이지에 로그인 안 하고 들어오면 `/login`으로 튕기는 것 = 미들웨어 인증 가드의 정확한 동작(`redirect`). 미들웨어가 없으면 페이지마다 로그인 체크 코드를 복붙해야 하는데, `matcher: ['/dashboard/:path*']` 하나로 그 구역 **전 페이지**를 한꺼번에 지킨다. 이게 실무 최대 이득.

(구분: "비번 틀림 = 로그인 실패"는 로그인 API가 판단. 미들웨어는 "통행증(쿠키) 있냐 없냐"만 본다.)

### 미들웨어 개념은 Next에서 나온 게 아니다
| 진영 | 이름 |
|---|---|
| 자바 서블릿(2000년대 초) | Filter |
| Spring | Interceptor / Security |
| Express(Node) | Middleware |
| Django / Rails | Middleware / Rack Middleware |
| Next.js(2020~) | Middleware |

"중간에서 요청을 가로채 처리한다"는 발상은 자바 Filter 시절부터 있던 오래된 것이고, **"미들웨어"라는 단어**는 Express(Node)가 웹에 퍼뜨렸다. Next는 그 이름과 개념을 물려받아 자기 식(`middleware.ts`, Edge 실행)으로 구현한 후발주자다.

## 아직 헷갈리는 것 / 다음에 볼 것
- `NextResponse.next()`가 내부적으로 "통과"를 어떻게 처리하는지.
- 쿠키 기반 인증 전체 흐름(로그인 → 쿠키 발급 → 미들웨어 검사 → 갱신).
- 미들웨어 인증 vs 서버 컴포넌트/route에서의 인증, 실무에선 어떻게 나눠 쓰나.

## 한 줄 요약
미들웨어(`middleware.ts`, 루트 하나)는 요청이 목적지에 닿기 전에 가로채는 **검문소**로, `NextResponse`로 통과(`next`)·차단(`redirect`)·우회(`rewrite`)를 결정하고 `matcher`로 적용 경로를 좁힌다 — 표준이 아닌 `NextRequest`/`NextResponse`를 쓰는 건 rewrite·next·쿠키 같은 기능이 거기에만 있어서고, 이건 Next가 만든 개념이 아니라 자바 Filter 시절부터 있던 걸 Express가 "미들웨어"로 대중화한 것이다(Spring판 짝 = Filter/Interceptor).
