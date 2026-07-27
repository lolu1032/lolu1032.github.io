---
title: "async와 await 차이 — fetch로 배우는 자바스크립트 비동기"
date: 2026-07-28 02:11:15 +0900
categories: [공부, JavaScript]
tags: [javascript, async, await, fetch, promise]
---

## 오늘 공부한 것
Next.js에서 fetch(백엔드 API 요청)를 읽다가 `async`/`await`가 궁금해졌다. "둘의 차이가 뭐냐"부터 시작했는데, 정리하니 애초에 **차이(경쟁)가 아니라 한 세트**였다. Spring(자바) 배경이라 blocking/non-blocking 비교로 이해했다.

## 내가 가졌던 질문
- `async`와 `await`의 차이가 뭔가
- Next에선 async/await만 쓰나? 이게 Next 전용인가
- 다른 방법은 없나

## 정리 (Q&A 핵심)

### async ↔ await는 차이가 아니라 짝꿍
"차이"로 물었는데, 이 둘은 둘 중 하나를 고르는 게 아니라 **함께 쓰는 짝**이다.

```
async = 함수에 붙이는 "선언" → "이 함수 안에서 기다리기(await)를 쓸 거야"
await = 실제 "동작"        → "이 Promise 끝날 때까지 기다렸다 결과 꺼내"
```

- `async` 함수는 **항상 Promise를 반환**한다.
- `await`는 **`async` 함수 안에서만** 쓸 수 있다. (서로가 필요해서 짝)
- 비유: `async`는 "기다림을 써도 되는 방"을 여는 것, `await`는 그 방에서 실제로 기다리는 행동.

### fetch로 보면 이해된다
```js
async function getUser() {
  const res  = await fetch('/api/user');  // ① 네트워크 응답 올 때까지 기다림
  const data = await res.json();          // ② 파싱 끝날 때까지 기다림
  return data;
}
```
`await`가 두 번인 이유: `fetch`도 Promise를 주고(네트워크가 시간이 걸림), `res.json()`도 Promise를 준다(파싱도 시간). 각각 기다려야 실제 값이 나온다.

### await 없으면 "약속"만 받는다
```js
const res = fetch('/api/user');  // await 안 붙임
console.log(res);  // Promise { <pending> }  ← 응답이 아니라 "약속"이 찍힘
```
`await`를 빼면 기다리지 않고 다음 줄로 가서, 손에 쥐는 건 아직 안 끝난 **Promise**다. `await`가 Promise를 **실제 값**으로 바꿔준다.

### 다른 방법도 있다 — `.then()`
async/await는 유일한 방법이 아니다. 같은 Promise를 다르게 쓰는 `.then()`도 있다.

```js
// .then 방식 — 아래 async/await와 완전히 같은 동작
fetch('/api/user')
  .then(res => res.json())
  .then(data => console.log(data));
```
밑바닥은 똑같은 Promise고, async/await가 그 위에 읽기 쉽게 얹은 문법일 뿐. 요즘은 async/await를 선호하지만 `.then()`도 여전히 많이 쓰인다.

### async/await는 Next 것이 아니라 JS 표준
Next가 만든 게 아니다. **ES2017부터의 자바스크립트 표준 문법**이라 브라우저·Node·React·Next 어디서나 똑같다. (미들웨어는 Next 거였지만, async/await는 JS 공용.)

### 왜 필요한가 — 자바(Spring)와 비교
이게 자바 배경과 결정적으로 다른 지점.

| | Java/Spring | JavaScript |
|---|---|---|
| I/O(API·DB) 기본 | **동기(blocking)** — 알아서 기다림 | **비동기(non-blocking)** — 안 기다리고 넘어감 |
| 결과 받기 | `String r = restTemplate.get(...)` 바로 값 | `const r = await fetch(...)` 기다려야 값 |

- Java는 `restTemplate.getForObject()` 하면 그 줄에서 **알아서 멈춰 기다렸다가** 값을 준다. 그래서 async를 딱히 의식 안 한다.
- JS는 fetch 같은 I/O가 **기본이 "안 기다리고 넘어감"**이라, `await`로 "여기선 기다려"를 직접 표시해야 한다.

즉 async/await는 JS의 "안 기다리는" 성질을, **자바처럼 "기다렸다 다음 줄"로 읽히게** 만드는 장치다. 대신 전체를 멈추는 게 아니라 그 함수만 기다려서 non-blocking 이점은 유지.

### Next에서 특별한 점 — 서버 컴포넌트가 통째로 async
Next(App Router) **서버 컴포넌트**는 컴포넌트 함수 자체에 async를 붙일 수 있다.

```jsx
// 서버 컴포넌트 — 컴포넌트가 통째로 async!
export default async function Page() {
  const res  = await fetch('...');
  const data = await res.json();
  return <div>{data.title}</div>;   // 데이터로 바로 화면
}
```
- 서버에서 도는 서버 컴포넌트라서 가능.
- **클라이언트 컴포넌트(`'use client'`)는 안 된다** — 컴포넌트를 async로 못 만든다. 대신 `useEffect` 안에서 fetch 하거나 SWR·React Query 같은 라이브러리로 처리.

## 아직 헷갈리는 것 / 다음에 볼 것
- `.then()` vs `async/await` 언제 뭘 쓰나 (기준).
- 서버 컴포넌트 fetch vs 클라 fetch — 어디서 데이터 가져오는 게 나은가.
- fetch 에러 처리(`try/catch`).

## 한 줄 요약
`async`와 `await`는 차이(경쟁)가 아니라 **짝**이다 — `async`는 "이 함수 안에서 기다림을 쓴다"는 선언(항상 Promise 반환)이고 `await`는 Promise가 끝날 때까지 기다렸다 실제 값을 꺼내는 동작으로, `await`는 `async` 함수 안에서만 쓸 수 있다. 이건 Next 전용이 아니라 JS 표준이고(`.then()`이라는 다른 방법도 있다), 자바가 I/O를 기본 동기로 알아서 기다리는 것과 달리 JS는 기본 비동기라 `await`로 "여기 기다려"를 직접 찍어주는 것이다.
