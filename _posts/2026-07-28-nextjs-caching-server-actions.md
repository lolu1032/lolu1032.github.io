---
title: "Next.js 캐싱과 서버 액션 — force-cache/force-static, revalidate, use server"
date: 2026-07-28 02:12:00 +0900
categories: [공부, Next.js]
tags: [nextjs, cache, revalidate, server-action, isr]
---

## 오늘 공부한 것
fetch를 읽다가 캐싱으로 넘어갔다. `force-cache`/`force-static`이 둘 다 캐싱 같은데 뭐가 다른지, `revalidate`와 온디맨드 재검증은 뭔지, 그리고 예시에 나온 `'use server'`(서버 액션)까지 정리. 반복해서 걸린 오해는 "이거 결국 DB까지 가봐야 아는 거 아냐?"였는데, 아니었다.

## 내가 가졌던 질문
- `force-cache`와 `force-static` 차이 (둘 다 캐싱 아닌가)
- `revalidate`는 스케줄러처럼 1시간마다 도는 건가
- "요청이 왔는지", "데이터가 바뀌었는지"는 DB를 가봐야 아는 것 아닌가
- 온디맨드 재검증이 뭔가, 많이 쓰나
- `'use server'`가 "이제부터 서버다"라는 뜻인가

## 정리 (Q&A 핵심)

### force-cache vs force-static — 점 vs 면
둘 다 캐싱은 맞는데 **레벨(단위)이 다르다.**

| | `force-cache` | `force-static` |
|---|---|---|
| 어디에 | **fetch 옵션** | **라우트 세그먼트 export** |
| 문법 | `fetch(url, { cache: 'force-cache' })` | `export const dynamic = 'force-static'` |
| 대상 | **fetch 요청 하나**의 응답 | **라우트(페이지/route.ts) 전체** |
| 레벨 | 데이터 (작은 단위, 점) | 렌더링 (큰 단위, 면) |

```js
export const dynamic = 'force-static';   // ← 이 라우트 "전체"를 정적으로 (면)

export default async function Page() {
  const a = await fetch(urlA, { cache: 'force-cache' });  // ← 이 fetch "하나"만 캐싱 (점)
  const b = await fetch(urlB);
  return <div>...</div>;
}
```

`force-static`은 더 큰 망치다. 붙이면 안의 fetch들도 자동으로 다 캐시되고, `cookies()`·`headers()` 같은 동적 기능도 막힌다(요청마다 달라지면 정적이 안 되니까). `force-cache`는 딱 그 fetch 하나만 건드린다.

### force-cache의 "데이터"는 DB가 아니라 fetch 응답
"페이지냐 데이터냐"의 축은 맞는데, force-cache의 "데이터"는 **`fetch`로 부른 API의 응답**이다. DB 자체가 아니다.

```js
const data = await fetch('https://api.../posts', { cache: 'force-cache' });
//                       └ 이 API 호출의 "응답"을 캐싱 (그 뒤 DB는 force-cache가 모름)
```
- fetch 없이 **DB를 직접 조회**하면(예: Prisma) force-cache가 **안 걸린다**. 그건 `unstable_cache`, React `cache()` 등 다른 도구를 써야 한다.

### revalidate는 스케줄러가 아니다 — 방문이 트리거
`revalidate = 3600`(1시간)은 타이머로 백그라운드에서 도는 게 아니라, **시간이 지난 뒤 요청이 들어왔을 때** 갱신되는 lazy 방식(ISR)이다.

```
① 1시간 안 지남 + 요청     → 캐시 그대로 응답 (fresh)
② 1시간 지난 후 첫 요청     → ⓐ 일단 "낡은 캐시"를 응답 (사용자 안 기다림)
                            ⓑ 동시에 백그라운드에서 새로 fetch → 캐시 교체
                            ⓒ "다음" 요청부터 새 캐시
③ 1시간 지났는데 아무도 안 옴 → 갱신 안 일어남 (요청이 트리거)
```

반전: ②를 트리거한 그 사람은 보통 **낡은 버전**을 받고, 새 데이터는 **다음 방문자부터** 본다(stale-while-revalidate). 비유하면 우유 유통기한 — 기한이 지나도 냉장고를 안 열면(요청 없으면) 그대로고, 열었을 때 새 걸로 교체하되 지금 여는 사람은 일단 있던 걸 받는다.

여기서 "요청"은 **GET API 요청이 아니라 사용자가 그 페이지에 접속한 것**이다.
```
바깥 요청 (트리거)  : 사용자(브라우저) → 페이지 접속   ← "요청 왔다"
안쪽 요청 (갱신 동작): Next 서버 → GET API 다시 fetch  ← 갱신할 때 서버가 함
```

revalidate도 두 군데에 쓸 수 있는데, 이게 앞의 면/점 구분과 같다.
```js
export const revalidate = 3600;                  // 라우트 전체 (면)
fetch(url, { next: { revalidate: 3600 } });      // fetch 하나 (점)
```

### "요청/변경 여부는 DB를 가봐야 아나?" — 아니다
반복해서 걸린 오해. 판단은 서버/캐시 레벨에서 끝나고, DB는 실제로 데이터를 가져올 때만 등장한다.

```
① 사용자 요청 도착   → 서버가 받은 순간 "요청 왔다"를 그냥 앎 (DB 무관)
② 캐시 낡았나?       → 캐시에 박힌 "만든 시각"을 비교 (서버 메모리, DB 무관)
③ 낡았으면 그제서야   → fetch → API → 그 뒤 DB  ← DB는 여기서만 등장
```
"손님이 왔는지 알려고 창고를 뒤지지 않는다" — 문지기(서버)가 문에서 바로 안다.

### 온디맨드 재검증 — 감지가 아니라 통보
시간을 기다리는 대신 **데이터가 바뀐 순간 직접 "지금 갱신해"라고 명령**하는 방식.

```js
import { revalidatePath, revalidateTag } from 'next/cache';
revalidatePath('/blog');    // /blog 페이지 캐시 즉시 갱신
revalidateTag('posts');     // 'posts' 태그 붙은 fetch 다 갱신
```
```js
fetch('https://.../api/posts', { next: { tags: ['posts'] } });  // 태그는 fetch에 붙임
```

여기서도 "데이터 바뀐 걸 DB 가봐서 아냐?" 싶지만 **아니다. 감지(detect)가 아니라 통보(notify)다.**

```js
'use server';
export async function publishPost(data) {
  await db.post.create({ data });   // ① DB에 저장 (내가 직접 바꿈)
  revalidatePath('/blog');          // ② 바로 다음 줄에서 갱신 명령
}
```
데이터를 바꾸는 코드가 = 재검증을 부르는 코드다. 같은 함수 안에 나란히 있어서 "확인"할 필요가 없다. 냉장고를 감시하다 발견하는 게 아니라, **채운 사람이 채우면서 라벨을 붙이는 것.** webhook도 마찬가지 — 바꾼 주체(CMS)가 직접 알린다. Next가 DB를 폴링하는 메커니즘은 없다(시간 기반도 타임스탬프만 볼 뿐 DB 안 감).

| | 시간 기반 revalidate | 온디맨드 |
|---|---|---|
| 트리거 | 시간 경과 + 방문 | 데이터 변경 **이벤트** |
| 정확도 | 최대 N초 낡을 수 있음 | 바뀐 즉시 |
| 쓰는 곳 | 변경 시점 모르는 외부 데이터 | 변경 시점 아는 경우(발행/수정, CMS webhook) |

변경 시점을 알 수 있으면 온디맨드가 더 정확하고 효율적이라 많이 쓴다(특히 Headless CMS + webhook).

### 'use server'와 서버 액션 — "서버 컴포넌트"가 아니다
`'use server'`는 "이제부터 서버다"도, "서버 컴포넌트"도 아니다. **서버 액션(함수)** 표시다.

```
'use client' → 클라이언트 "컴포넌트"로 만듦   (컴포넌트 얘기)
'use server' → 서버 "액션(함수)"으로 만듦     (함수 얘기! 컴포넌트 아님)
```
서버 컴포넌트는 아무것도 안 붙이는 게 기본이다. `'use server'`가 알리는 것은 **"여기 함수들은 서버에서 도는 액션이야(클라가 불러도 됨)"**. 파일 맨 위에 붙이면 그 파일 함수 전부, 함수 안에 붙이면 그 함수만.

핵심 이득: `route.ts`+`fetch` 배관을 직접 안 짜도 **클라가 함수를 직접 호출**하면 Next가 뒤에서 API를 자동 생성한다.
```js
await publishPost(data);   // fetch('/api/...') 안 씀. Next가 HTTP를 자동 연결
```

주로 **변경(mutation)** 작업에 쓴다(생성·수정·삭제, 폼 제출) — 그래서 이름이 "액션". 읽기(GET)는 보통 서버 컴포넌트에서 직접 한다. 함수 호출이라 인자·반환 타입이 클라-서버 넘어 TypeScript로 체크되는 보너스도 있다.

### 서버 액션 ≈ Spring Controller (단, URL 없는)
프론트가 부르는 서버 진입점이라는 점에서 Spring Controller와 같다. 다만 부르는 방식이 다르다.

| | Spring Controller | Next 서버 액션 | Next route.ts |
|---|---|---|---|
| 계약 | 주소(URL) | **함수** | 주소(URL) |
| 부르는 법 | `fetch('/api/...')` | 함수 직접 호출 | `fetch('/api/...')` |
| 타입 검사 | 수동(DTO) | 자동(TS) | 수동 |

서버 액션은 "URL 없는 컨트롤러"인 셈. Spring Controller에 더 똑 닮은 건 오히려 URL 있는 `route.ts`(API 라우트)다. 외부·webhook이 부를 거면 URL 필요하니 `route.ts`, 내 프론트에서만 부를 거면 간편한 서버 액션.

## 아직 헷갈리는 것 / 다음에 볼 것
- `unstable_cache` / React `cache()`로 DB 직접 조회 캐싱하기.
- 서버 액션 + `<form action={}>` 실제 폼 처리 흐름.
- Next 15에서 fetch 기본 캐시 정책이 바뀐 부분.

## 한 줄 요약
`force-static`(라우트 전체=면)과 `force-cache`(fetch 응답 하나=점)는 캐싱 레벨이 다르고(force-cache의 데이터는 DB가 아니라 fetch 응답), `revalidate`는 스케줄러가 아니라 시간이 지난 뒤 사용자 방문이 트리거하는 lazy 갱신(stale-while-revalidate)이며, 온디맨드 재검증은 데이터를 바꾼 코드가 바로 옆에서 `revalidatePath`/`revalidateTag`로 통보하는 방식이다(감지 아님). 그리고 `'use server'`는 서버 컴포넌트가 아니라 서버 액션(함수) 표시로, `route.ts`+`fetch` 없이 클라가 직접 호출하는 "URL 없는 컨트롤러"다.
