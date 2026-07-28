---
title: "SWR vs 서버 캐시(ISR) — stale-while-revalidate를 서버에서 하느냐 브라우저에서 하느냐"
date: 2026-07-29 01:15:31 +0900
categories: [공부, Next.js]
tags: [swr, isr, cache, revalidate, seo]
---

## 오늘 공부한 것
내 프로젝트(workb)가 SWR을 쓰길래, SWR이 뭔지 파다가 어제 배운 캐시(force-cache/revalidate)와 계속 이어졌다. "SWR은 서버 캐시를 합친 건가?"에서 시작해 서버 캐시(ISR) vs SWR, SEO, 하이브리드 패턴까지 정리.

## 내가 가졌던 질문
- SWR이 `force-cache` + `revalidate`를 합친 건가
- "브라우저에서 캐시/갱신"이면 새로 접속해야 갱신되나
- 화면이 깜빡이면 그게 SWR 갱신인가
- 커뮤니티처럼 실시간으로 글이 뜨면 깜빡임 없이 최신이 보이나
- "내 글은 즉시, 남 글은 몇 초 뒤"가 SWR인가
- `mutate()`가 서버의 `revalidatePath` 같은 건가
- refine에서 쓴 ISR 캐시와 SWR의 차이
- ISR이 SEO랑 무슨 상관인가
- 로그인 전엔 ISR, 후엔 SWR이 맞나

## 정리 (Q&A 핵심)

### SWR은 "합친 것"이 아니라 "같은 전략을 다른 층에서"
SWR의 동작("캐시된 거 먼저 주고, 뒤에서 최신 받아 업데이트")은 그 이름 그대로 **stale-while-revalidate**다. 어제 배운 Next `revalidate`도 같은 패턴이었다. 그래서 뿌리는 같은데, **작동하는 층이 다르다.**

```
Next force-cache/revalidate → 서버에서 캐시하고 서버에서 갱신   (서버 층)
SWR                         → 브라우저에서 캐시하고 브라우저에서 갱신 (클라 층)
```

즉 "force-cache + revalidate를 합쳐서 SWR"이 아니라, 같은 stale-while-revalidate를 **서버는 Next 캐시로, 클라는 SWR로 각각 구현**한 사촌 관계.

### "새로 접속하면 갱신"이 아니다 — 풀 리로드는 리셋
SWR 캐시는 **브라우저 메모리** 캐시라 탭이 살아있는 동안만 존재한다.

- **F5 / 새로 접속** → 메모리 날아감 → 캐시도 사라짐 → 처음부터 새로 fetch(콜드 스타트). 갱신이 아니라 **리셋**.
- 진짜 "갱신"은 **탭이 열린 채** 일어나는 이벤트가 트리거:
  - **탭 재포커스**(`revalidateOnFocus`) — 다른 탭 갔다 돌아올 때 (제일 특징적)
  - 컴포넌트 재마운트 / 네트워크 재연결 / 인터벌(`refreshInterval`) / 수동 `mutate()`

어제 배운 서버 캐시는 "접속(요청)이 트리거"였는데, SWR은 반대로 새 접속이 리셋이고 갱신은 in-tab 이벤트다. 층이 달라서 반대.

### 화면 깜빡임 ≠ SWR 갱신 (오히려 반대)
정상적인 SWR 갱신은 **안 깜빡인다.** 낡은 데이터를 계속 띄운 채 뒤에서 바꾸고, 바뀐 값만 스르륵 교체하기 때문. 깜빡인다면 대개 다른 원인:

- 캐시 없는 **첫 로드**(로딩 → 데이터)
- 컴포넌트 **재마운트**(어제 배운 template 재마운트 깜빡)
- 로딩 UI를 재검증(`isValidating`)에도 띄우게 짠 경우(안티패턴)

SWR 상태로 구분: `data`는 갱신 중에도 낡은 값을 유지(→ 안 깜빡), `isLoading`은 캐시 없는 첫 로드일 때만 true.

### 실시간 커뮤니티 — 깜빡임은 없지만 SWR은 "pull"이다
"글이 실시간으로 뜨면 깜빡임 없이 최신이 보이나?" → 깜빡임 없이 부드럽게 갱신되는 건 맞다. 하지만 SWR은 **push가 아니라 pull**이다.

| 상황 | SWR로 | 방법 |
|---|---|---|
| 내가 글 씀 | ✅ 즉시 | 저장 후 `mutate()` |
| 남이 글 씀 | △ 조건부 | `refreshInterval` 폴링(N초) → 거의 실시간 |

- 누가 쓰는 **순간** 즉시 뜨는 진짜 실시간은 **WebSocket/SSE(서버 push)**가 필요하다. (workb 인프라의 Centrifugo가 그 역할.) 보통 WebSocket 신호를 받아 `mutate()`로 갱신하는 식으로 조합.

### "내 글 즉시 / 남 글 몇 초 턴"의 정체
이 증상이 딱 그 패턴의 지문이다.
```
내가 씀  → mutate()로 즉시 갱신                  (0초)
남이 씀  → 다음 폴링/재검증 때 반영              (몇 초 턴)
```
정확히는 "SWR이라서"보다 **"pull(폴링) 방식이라서"** 생기는 공통 증상(SWR이 대표). 남 글도 즉시 뜨면 WebSocket(push). 내 글이 체감 0초면 **낙관적 업데이트**(서버 응답 안 기다리고 화면부터)일 수 있다.

### mutate()는 SWR판 revalidatePath — 층별 대칭
서버든 클라든 "저장 / 온디맨드 갱신 / 시간 갱신"이 똑같이 대응된다.

| | 서버 (Next 캐시) | 클라 (SWR) |
|---|---|---|
| 캐시 저장 | `force-cache`/`revalidate` | `useSWR` (메모리) |
| **온디맨드 갱신(통보)** | **`revalidatePath()`/`revalidateTag()`** | **`mutate()`** |
| 시간 기반 | `revalidate = N` | `refreshInterval: N` |

둘 다 "지금 이 캐시 갱신해"라는 통보(감지가 아님). `mutate`는 한발 더 나가 "다시 fetch"뿐 아니라 `mutate(key, newData)`로 **화면 즉시 세팅(낙관적)**까지 된다.

### refine(서버 캐시/ISR) vs SWR — 또 서버냐 브라우저냐
refine 프로젝트(refinedv-v2)가 실제로 쓴 것:
```ts
export const getBoardPosts = unstable_cache(
  async (...) => { /* DB 조회 */ },
  [...],
  { revalidate: 600, tags: ["board-posts"] }   // 10분 시간캐시 + 태그 온디맨드
);
```
- `unstable_cache` = 서버에서 DB 결과 캐시(어제 "DB 직접 조회는 force-cache 안 걸려서 이걸 쓴다"고 한 실물).
- `revalidate: 600` + `revalidateTag` = 시간 + 온디맨드. 전부 **서버 층**.

| | refine 캐시 (서버, ISR 계열) | SWR (브라우저) |
|---|---|---|
| 도는 곳 | 서버 | 브라우저 |
| 누가 새 걸 보나 | **다음 방문자**부터 | **지금 보는 나**의 화면 자동 갱신 |
| 화면 자동 변함 | ❌ (재방문/새로고침) | ✅ (리렌더) |
| SEO | ✅ 완성 HTML | ❌ JS 후 데이터 |
| 잘 맞는 것 | 공용·잘 안 바뀜(게시판 목록) | 유저별·상호작용(근태) |

### ISR이 SEO에 좋은 이유 = 완성 HTML을 서버가 만들어서
ISR 자체의 마법이 아니라, **서버가 내용이 든 완성 HTML을 만들어 봇이 바로 읽을 수 있어서**다. 예전 CSR/SSR/SSG에서 배운 그 축과 같다.

```
ISR = SSG 계열(서버가 완성 HTML)  → 봇이 읽음  → SEO 좋음
SWR = CSR 계열(브라우저가 JS로 채움) → 봇이 빈 HTML 받음 → SEO 약함
```
- 봇이 받는 ISR HTML엔 `<li>글 제목</li>`이 이미 박혀 있고, SWR은 `<div id="board"></div>`처럼 비어 있다(JS 실행 후 채워짐).
- 뉘앙스: 요즘 구글봇은 JS도 어느 정도 돌리지만 느리고 불안정하며, 카톡 미리보기·타 봇은 JS를 안 돌린다. 확실한 SEO는 완성 HTML.

### "로그인 전 ISR / 후 SWR"? — 기준은 timing이 아니다
대충 맞는 프록시지만 진짜 기준은 **공개/SEO/공용이냐 vs 유저별/상호작용/실시간이냐**다(로그인 여부 무관). 로그인 후에도 공개 게시판 목록은 서버렌더(SEO)일 수 있고, 로그인 전에도 실시간 위젯은 SWR일 수 있다.

다만 "초반 서버 → 그 뒤 클라"라는 직관은 **세션이 아니라 한 페이지 안에서 보면 정석 하이브리드**다.
```tsx
// 서버 컴포넌트 — 초기 데이터는 서버에서 (SEO + 즉시 표시)
const initial = await getBoardPosts();
return <BoardList fallback={initial} />;

// 클라 컴포넌트 — SWR이 이어받아 갱신
'use client';
const { data } = useSWR('/api/posts', fetcher, { fallbackData: fallback });
```
첫 HTML은 서버가 완성(SEO), 그다음부터 SWR이 `fallbackData`로 이어받아 실시간 갱신. 게시판이면 공개 목록은 이 하이브리드, 쓰기·실시간은 SWR/서버액션.

## 아직 헷갈리는 것 / 다음에 볼 것
- SWR `fallbackData`에 서버 데이터 넘기는 실제 코드(서버 컴포넌트 → 클라).
- WebSocket(Centrifugo) 신호 → `mutate()` 연동 흐름.
- SWR vs React Query(내 프로젝트 다수가 이걸 씀) 차이.

## 한 줄 요약
SWR은 `force-cache`+`revalidate`를 합친 게 아니라 **같은 stale-while-revalidate를 브라우저 층에서 구현**한 것이다 — 서버 캐시(ISR/refine의 unstable_cache)는 서버에서 완성 HTML을 갱신해 "다음 방문자부터 최신 + SEO 좋음"이고, SWR은 브라우저에서 데이터를 갱신해 "지금 보는 화면이 저절로 바뀜 + 실시간감(단 pull이라 남 글은 폴링, SEO 약함)". `mutate()`는 서버의 `revalidatePath`에 대응하는 클라판 통보이고, 실무에선 **서버로 첫 HTML(SEO) → SWR로 이어받아 갱신**하는 하이브리드로 상황에 맞게 나눠 쓴다.
