---
title: "Next.js API 라우트와 서버 컴포넌트 — request/헤더/body 다루기"
date: 2026-07-24 01:51:55 +0900
categories: [공부, Next.js]
tags: [nextjs, api-route, 서버컴포넌트, request, header]
---

## 오늘 공부한 것
API 라우트(route.ts)를 배우면서 request/response, 헤더 읽기, body 파싱, 캐싱, 그리고 서버 컴포넌트가 뭔지까지 정리. Spring Boot 경험이 있어서 그쪽 개념에 매핑하며 이해함.

## 내가 가졌던 질문
- API 라우트는 폴더 이름(`api`)이나 파일 이름(`signin`)이 기준인가
- `Request` vs `NextRequest`, `Response` vs `NextResponse` 차이
- request는 Spring의 DTO 같은 건가
- `force-static`으로 GET 캐싱
- 헤더 다루기가 중요한가 (Referer 예시)
- 서버 컴포넌트가 뭔가, 우리가 만든 page.tsx는 서버/클라 중 뭔가
- `json()` vs `formData()` 차이

## 정리

### API 라우트 — 화면 대신 JSON 반환
`page.tsx`가 화면(HTML)을 반환한다면, API 라우트는 데이터(JSON)를 반환하는 **백엔드 엔드포인트**.

- **Pages Router**: `pages/api/` 폴더 안이면 API (`pages/api/signin.ts` → `/api/signin`)
- **App Router**: 폴더명이 아니라 **`route.ts` 파일**이 기준. 관례상 `api/`에 둠.
- `signin` 같은 이름 때문이 아니라 **위치/파일**이 API 인식 기준.
- `GET`/`POST`/`PUT`/`DELETE` **HTTP 메서드 이름으로 export**해서 처리(약속된 이름).

```ts
// app/api/signin/route.ts
export async function POST(request: Request) {
  const { email, password } = await request.json();
  return Response.json({ ok: true });
}
```

### Request vs NextRequest / Response vs NextResponse
`NextRequest`는 `Request`를 상속해 편의 기능을 추가한 확장판.

| 하는 일 | `Request` | `NextRequest` |
|---|---|---|
| **body 꺼내기** | `await request.json()` | `await request.json()` **(동일!)** |
| 쿼리 | `new URL(req.url).searchParams` | `req.nextUrl.searchParams` |
| 쿠키 | 헤더에서 직접 파싱 | `req.cookies.get()` |
| IP·지역 | 없음 | `req.ip`, `req.geo` |

- **핵심**: body 꺼내기(`json()`)는 둘 다 똑같다. 차이는 body가 아니라 **쿼리·쿠키·IP 같은 다른 정보 접근**에서 남.
- `NextResponse`는 `redirect`, `rewrite`, `cookies.set` 등이 편함. **미들웨어에선 NextResponse 거의 필수**.
- body만 다루면 표준으로 충분, 쿠키·리다이렉트 필요하면 Next 버전.

### request와 body — Spring 매핑
- `Request` ≈ Spring의 `HttpServletRequest`(요청 전체 봉투). **DTO가 아님.**
- **DTO는 `request.json()`으로 꺼낸 body** — 그 모양을 `interface`로 정의.
- 큰 차이: Spring은 `@RequestBody`로 **자동 변환**+`@Valid` **자동 검증**. Next는 `request.json()`으로 **수동**이고, `as SigninDto`는 **타입 힌트일 뿐 런타임 검증 안 함**(앞에서 배운 타입 단언 `as` 그대로 — TS 눈만 속임). 실무에선 **zod**로 검증(= Spring의 `@Valid` 역할).

| Spring | Next.js |
|---|---|
| `HttpServletRequest` | `Request`/`NextRequest` |
| `@RequestBody SigninDto` | `await request.json()` + 타입 |
| `@Valid` 검증 | zod `.parse()` |

### json() vs formData()
body에 담긴 **데이터 형식** 차이.

| | JSON | FormData |
|---|---|---|
| 꺼내기 | `data.email`(객체) | `form.get('email')` |
| **파일** | ❌ 불가 | ✅ 가능 |
| 용도 | API 통신(로그인·댓글) | HTML 폼, **파일 업로드** |

- **JSON은 파일을 못 담음** → 이미지·첨부파일 업로드는 **FormData 필수**.
- Spring 매핑: `json()` ≈ `@RequestBody`, `formData()` ≈ `@RequestParam`/`MultipartFile`.

### force-static — GET 캐싱 (API판 SSG)
`export const dynamic = 'force-static'`을 route.ts에 붙이면 GET을 **빌드 때 1번 실행해 결과를 굳힘**(요청마다 실행 X). 원리가 SSG랑 같음.

- 장점: 빠름 + DB 부담 없음 + CDN 캐싱 (SSG 장점 그대로)
- 함정: **데이터가 안 바뀜**(빌드 값 고정), **요청별 응답 못 함**(쿠키·쿼리 못 씀), **GET만**
- 쓸 곳: 공지·약관·카테고리 같은 안 바뀌는 공용 데이터. 실시간·유저별엔 금지.
- 절충: `export const revalidate = 3600`(ISR) — N초마다 갱신. 가끔 바뀌는 데이터용.

### 헤더 다루기 (중요도는 상황따라)
헤더 = 요청/응답의 부가정보(인증 토큰·Content-Type·쿠키·캐시·보안). 화면만 만들 땐 거의 안 건드리지만, **인증/API/보안 붙으면 필수**.

- 요청 헤더 읽기: `request.headers.get('authorization')`
- 응답 헤더/쿠키: `NextResponse` + `res.cookies.set(...)`(= Set-Cookie 헤더)
- **Referer**: "이 요청이 직전에 어느 페이지에서 왔는지" 알려주는 헤더. 유입 경로 분석·출처 체크용. 단 **null일 수 있고 위조 가능** → 편의·분석엔 OK, 진짜 보안엔 의존 X. 철자 함정: `referer`(r 하나, 표준 오타).

### 서버 컴포넌트 vs 클라이언트 컴포넌트 (App Router 핵심)
컴포넌트가 **어디서 실행되냐**로 갈림.

| | 서버 컴포넌트 | 클라이언트 컴포넌트 |
|---|---|---|
| 표시 | 기본(안 붙임) | `'use client'` |
| 실행 위치 | 서버 | 브라우저 |
| 가능 | DB·헤더 직접 접근 | useState·onClick·useEffect |
| 불가 | 상태·클릭·이펙트 | DB·헤더 직접 접근 |

- **우리가 만든 page.tsx/layout.tsx = 기본 서버 컴포넌트.** `'use client'` 붙은 것만 클라이언트(FeedLayout이 useState 써서 예외였음).
- 서버 컴포넌트엔 **request 파라미터가 없음**(`{ params, searchParams }`만 받음) → 헤더는 `next/headers`의 `headers()` 함수로 가져옴. `await` 붙는 건 비동기라서. 클라이언트 컴포넌트에선 불가(서버 전용).

```ts
// API 라우트: request가 손에 있음 → 직접
request.headers.get('referer');
// 서버 컴포넌트: request 없음 → 함수로 가져옴
const h = await headers(); h.get('referer');
```

### "request 없는데 어느 요청 헤더를 가져오나?" (중요 통찰)
`headers()`는 **"지금 이 컴포넌트를 실행시킨 바로 그 요청"**의 헤더를 가져온다. 여러 요청이 동시에 와도 안 섞이는 이유 = **요청마다 독립된 실행**이 일어나기 때문. A 요청 실행 중엔 A 헤더, B 실행 중엔 B 헤더.

- 사실 서버 컴포넌트에도 request가 "숨어" 있고, `headers()`가 그 숨은 요청에 접근하는 창구. API 라우트(직접 받음)와 본질은 같음.
- 주의: "요청마다 개별"이라 **전역 변수에 요청 데이터를 담으면 안 됨**(요청 간 공유돼 섞임). 지역 변수·`headers()`는 실행 안에 머물러 안전.

## 아직 헷갈리는 것 / 다음에 볼 것
- 미들웨어(`middleware.ts`)로 로그인 체크·리다이렉트 (NextResponse 실전)
- 서버 컴포넌트 안에 클라이언트 컴포넌트 끼워넣기(경계 나누기)
- zod로 body 검증 실제 코드
- 쿠키 기반 인증 전체 흐름

## 한 줄 요약
API 라우트는 `route.ts`에서 JSON을 반환하는 백엔드(메서드 이름으로 export), request는 Spring의 HttpServletRequest에 해당하고 body는 `json()`/`formData()`로 꺼내며(파일은 FormData), 헤더는 request가 있으면 `request.headers` 없으면(서버 컴포넌트) `headers()`로 읽는다 — 그리고 우리가 만든 page.tsx는 기본 서버 컴포넌트라 요청마다 독립 실행된다.
