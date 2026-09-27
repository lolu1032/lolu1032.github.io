---
title: "props와 children — 클라 컴포넌트 안에 서버 컴포넌트 넣기"
date: 2026-08-04 01:44:10 +0900
categories: [공부, React]
tags: [react, props, children, nextjs, fragment]
---

## 오늘 공부한 것
렌더링을 보다 props가 이해 안 돼서 처음부터 다시 잡았다. props가 어떻게 동작하는지 → children이 뭔지 → 클라 컴포넌트가 children으로 서버 컴포넌트를 받는 패턴(책 9장 예제)까지 이어졌다.

## 내가 가졌던 질문
- props가 대체 어떻게 동작하나
- `props.chicken`처럼 꺼내면 넘긴 값이 변형되나
- `<ClientComponent><ServerComponent/></ClientComponent>`면 `props.children`이 ServerComponent인가
- 왜 ServerComponent가 구멍에 들어가나, ClientComponent가 들어갈 순 없나
- `<> </>`(Fragment)가 ClientComponent인가
- 이 패턴에서 버튼 말고 우리(children)는 뭘 하나

## 정리 (Q&A 핵심)

### props = 부모가 자식(=함수)에게 넘기는 인자
컴포넌트는 함수고, props는 그 함수의 매개변수다.

```jsx
function Welcome({ name }) {     // 자식: props에서 name 꺼냄
  return <h1>안녕, {name}</h1>;
}
<Welcome name="박" />           // 부모: 속성으로 값 넘김
```

`<Welcome name="박" />`은 사실상 `Welcome({ name: "박" })` 함수 호출이다. JSX 속성이 객체로 묶여 함수 첫 인자(props)로 들어온다. 규칙 두 개: **부모→자식 단방향**, **읽기 전용**(자식이 props 못 바꿈). Java 메서드에 파라미터 넘기는 것과 같다.

### props는 값을 변형하지 않는다 (라벨 붙인 상자)
"`props.chicken`이 병아리냐?" → 아니다. 넣은 🐔 그대로 나온다. props는 라벨 붙여 운반만 하지 내용을 안 바꾼다.
- 이름(`chicken`)은 내가 정하는 임의 라벨. 넘길 때 `chicken={...}`이면 `props.chicken`으로 꺼낸다. 철자·대소문자가 같아야 함.

### children = 여닫는 태그 "사이"에 놓인 것
`children`은 특별한 prop이다. 태그 사이 내용이 자동으로 `children`으로 들어간다.

```jsx
<Welcome>이 안의 내용</Welcome>   // "이 안의 내용"이 children
function Welcome({ children }) {
  return <div>{children}</div>;
}
```

중요한 건 **"무슨 컴포넌트냐"가 아니라 "어디에 썼냐(위치)"**로 정해진다는 것. 그래서 순서를 바꾸면 무엇이든 children이 될 수 있다.

### 책 예제 — 클라 껍데기 + 서버 알맹이
```tsx
// ClientComponent.tsx
'use client';                                  // useState 쓰니 필수
import { useState } from 'react';

interface ClientComponentProps {
  children: React.ReactNode;                   // children의 타입(렌더 가능한 아무거나)
}

function ClientComponent(props: ClientComponentProps) {
  const [count, setCount] = useState(0);
  return (
    <>                                         {/* 여러 개를 하나로 묶는 투명 포장지 */}
      <button onClick={() => setCount(count + 1)}>{count}</button>  {/* 버튼 = 상호작용 */}
      {props.children}                         {/* 구멍 = 밖에서 넣은 게 여기로 */}
    </>
  );
}
export default ClientComponent;
```
```tsx
// page.tsx (서버 컴포넌트)
function Page() {
  return (
    <ClientComponent>
      <ServerComponent />   {/* 이게 props.children으로 들어감 */}
    </ClientComponent>
  );
}
```

`<ClientComponent>`는 버튼이 아니라 **"버튼 + 구멍"**을 그리는 상자다. `{props.children}` 자리에 `<ServerComponent/>`가 끼워져 버튼과 나란히 그려진다.

```
┌──────────────────┐
│  [버튼: count]    │  ← 자기 버튼 (상호작용)
│  게시글 제목       │  ← props.children (ServerComponent = 서버가 가져온 내용)
└──────────────────┘
```

### 왜 이 방향(클라 바깥 + 서버 안)인가
children은 위치로 정해지니 ClientComponent도 사이에 두면 children이 된다. 근데 책이 굳이 이 방향을 쓴 이유:

```
클라가 서버를 import ❌  →  children으로 넣는 우회로가 필요  ← 이 방향
서버가 클라를 import ✅  →  그냥 import로 되니 트릭 불필요    ← 반대 방향
```

클라 컴포넌트는 서버 컴포넌트를 import하면 그 코드가 브라우저 번들에 딸려가 "서버 전용"이 깨진다. 그래서 막혀 있다. 대신 **서버(Page)가 `<ServerComponent/>`를 미리 렌더하고, 그 결과만 클라의 구멍에 끼우면** 서버 컴포넌트가 서버 컴포넌트인 채로 클라 껍데기 안에 들어간다. ClientComponent는 children으로 뭐가 올지 모르고 그냥 렌더만 하니까 가능한 것.

이 덕에 상호작용 껍데기 하나 때문에 안쪽 전부를 클라로 만들 필요가 없다 — **껍데기만 클라, 알맹이는 서버**(DB 조회·JS 절감 유지).

### `<> </>` = Fragment (투명 포장지)
`<>`는 ClientComponent가 아니다. React return은 최상위가 하나여야 하는데 버튼과 children 둘을 그리고 싶어서, `<div>` 안 만들고 묶기만 하는 빈 껍데기. 화면에 아무것도 안 그리고 기능도 없다.

### 버튼 말고 "우리(children)"는 뭘 하나
버튼은 움직이는 담당, children(ServerComponent) 자리는 **보여주는 담당** — 게시글 제목·내용처럼 서버가 가져온 진짜 내용. 예제라 이름이 밋밋해서 아무것도 안 하는 것처럼 보일 뿐, 실제론 여기가 알맹이다.

## 아직 헷갈리는 것 / 다음에 볼 것
- `React.ReactNode` 외의 children 타입 좁히기(특정 요소만 받기).
- children 여러 개를 이름 붙여 받는 패턴(named slots).
- 서버 컴포넌트를 children으로 넘길 때 props 함께 넘기는 법.

## 한 줄 요약
props는 부모가 자식(=함수)에게 넘기는 인자(단방향·읽기전용, 값을 변형 안 함)이고, children은 여닫는 태그 "사이"에 놓인 것이 들어가는 특별한 prop이다 — 위치로 정해지므로, 클라 컴포넌트를 바깥에 두고 서버 컴포넌트를 사이에 넣으면 `{props.children}` 구멍에 서버 컴포넌트가 서버인 채로 끼워진다(클라는 서버를 import 못 하니 이게 유일한 우회로). 껍데기(버튼)는 클라, 알맹이(게시글)는 서버.
