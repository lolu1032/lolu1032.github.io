---
title: "Next.js 레이아웃·템플릿과 마운트 동작 (App Router)"
date: 2026-07-21 00:38:37 +0900
categories: [공부, Next.js]
tags: [nextjs, app-router, 레이아웃, template, 마운트]
---

## 오늘 공부한 것
App Router의 레이아웃을 파다 보니 루트 레이아웃 / 중첩 레이아웃 / template이 헷갈렸다. 실제 코드(루트 레이아웃, FeedLayout)를 보며 "마운트"가 뭔지까지 정리.

## 내가 가졌던 질문
- 루트 레이아웃이 뭐고, 왜 `<html>`/`<body>`가 거기 있나
- 특수 파일들(layout/template/error/loading/not-found/page)의 계층 그림이 뭔 소리인가
- template은 layout이랑 뭐가 다르고, "매번 초기화"가 어떤 경우인가
- 마운트가 정확히 뭔가
- 중첩 레이아웃(FeedLayout)이 특정 세그먼트 레이아웃 맞나

## 정리

### 루트 레이아웃 (app 최상위)
`app/layout.tsx` = App Router의 **필수 최상위 레이아웃**. 모든 페이지가 이걸 거친다.

- `<html>`/`<body>` 태그를 **여기서만** 포함(중첩 레이아웃엔 넣지 않음). HTML 문서엔 html/body가 하나씩만 있어야 하니 루트에서 한 번만.
- Pages Router의 `_app`(공통 래퍼) + `_document`(HTML 뼈대)를 **합친 것**. 그래서 App Router엔 `_document`가 따로 없고 `<html lang>`도 루트 레이아웃에서 처리.

실제 루트 레이아웃엔 이런 게 다 들어간다: `import './globals.css'`(전역 CSS는 최상위에서만), `export const metadata`(메타데이터), 폰트 설정, `<nav>`+`<Link>`(공통 네비), `{children}`(페이지 자리), 그리고 병렬 라우트 슬롯(`{trends}`,`{users}`).

### 중첩 레이아웃 (특정 세그먼트)
"공통의 범위가 층층이 다를 때" 쓴다. 전체 공통(헤더/푸터) 위에, 특정 섹션만의 공통 틀(사이드바 등)을 한 겹 더.

- **쓰는 경우**: 대시보드(사이드바), 문서(목차), 설정(탭) — 그 URL 섹션의 페이지들끼리만 공유하는 틀이 있을 때.
- 특정 경로에 `layout.tsx`를 두면 **그 경로의 모든 하위 경로에도** 적용. 예: `app/feed/layout.tsx` → `/feed`, `/feed/posts/1` 다 감쌈.
- App Router는 폴더 깊이대로 자동 중첩. (Pages Router는 `getLayout` 패턴으로 수동.)
- 중첩 레이아웃엔 `<html>`/`<body>`를 넣지 않는다(루트 전용).

```
[루트 레이아웃] nav + 슬롯 (항상)
  └─ [FeedLayout] (/feed 일 때만)
        └─ [Page] 실제 내용
```

### 특수 파일 계층 — 파일 이름만 만들면 자동으로 감싼다
App Router는 약속된 이름의 파일을 만들면 정해진 순서로 페이지를 감싼다(직접 Suspense/ErrorBoundary 안 짜도 됨).

```
<Layout>                              layout.js
  <Template>                          template.js
    <ErrorBoundary fallback={Error}>  error.js
      <Suspense fallback={Loading}>   loading.js
        <ErrorBoundary fallback={NotFound}> not-found.js
          <Page />                    page.js  (제일 안쪽)
```

바깥일수록 넓은 처리: `error`가 `loading`보다 바깥이라 로딩 중 에러도 잡고, `loading`이 `page`보다 바깥이라 데이터 로딩 중 Loading 표시.

### layout vs template
거의 같은데 딱 하나 다르다.

| | `layout` | `template` |
|---|---|---|
| 이동 시 | **유지**(리마운트 X, 상태 보존) | **매번 새로 생성**(재마운트, 상태 리셋) |
| 성능 | 좋음 | 매번 다시 그림 |
| 기본 | ✅ 대부분 | 특수 목적만 |

**template을 쓰는 경우(= 매번 초기화 필요)**:
1. 페이지 이동마다 **진입 애니메이션** 재생
2. 이동마다 **`useEffect` 재실행**(방문 기록 등)
3. 이동마다 **상태 초기화**(폼·입력 리셋)

원리: layout=인스턴스 유지(useEffect 안 다시 돎, 상태 유지), template=매번 재마운트(useEffect 다시 돎, 상태 리셋). 대부분은 layout이 기본이고, "매번 리셋"이 명확히 필요할 때만 template.

### 마운트란?
컴포넌트가 **화면(DOM)에 처음 나타나는 것**. 생명주기: 마운트(태어남) → 업데이트(삶) → 언마운트(사라짐). 비유하면 전구를 소켓에 끼우는(마운트)/빼는(언마운트) 것.

- `useEffect(() => {...}, [])` = **마운트 때 딱 1번** 실행.
- 실제 마운트는 즉시. 책의 "mounting... → (1초) → mounted."는 `setTimeout`으로 일부러 지연시켜 마운트를 눈에 보이게 한 데모.

**FeedLayout 데모의 의미** (layout 유지 증명):
```
/feed 진입         → mounting... → mounted.
/feed/posts/1 이동 → (유지) mounted.   ← 재마운트 안 함 (layout이라)
/feed → /search    → 언마운트 (사라짐)  ← 세그먼트 밖으로 나감
```
- 레이아웃은 **그 세그먼트 범위 안에 있는 동안** 유지, 벗어나면 언마운트. 루트 레이아웃만 어디서든 유지(최상위라 어느 경로든 그 안).
- 만약 template이었으면 이동마다 재마운트 → 매번 "mounting..."부터 반복(깜빡).

## 아직 헷갈리는 것 / 다음에 볼 것
- error.js / loading.js가 실제로 뜨는 흐름을 코드로 확인.
- 병렬 라우트 슬롯의 `default.js`가 새로고침(hard navigation) 때 뜨는 이유 더 파기.

## 한 줄 요약
루트 레이아웃(app 최상위, `<html>/<body>` 포함, `_app`+`_document` 합체)이 모든 페이지를 감싸고, 중첩 레이아웃은 특정 세그먼트에만 적용되며, 이들은 이동해도 마운트가 유지되지만 template은 이동마다 재마운트(상태 리셋)된다 — "유지가 필요하면 layout, 매번 리셋이 필요하면 template".
