---
title: "Next.js 메타데이터와 SEO — metadata·robots·sitemap·OpenGraph"
date: 2026-07-21 00:39:00 +0900
categories: [공부, Next.js]
tags: [nextjs, metadata, seo, opengraph, sitemap]
---

## 오늘 공부한 것
메타데이터/SEO 파일들을 읽는데 정적·동적 메타데이터, robots.txt, sitemap.xml, OpenGraph가 각각 뭔지 헷갈려서 정리.

## 내가 가졌던 질문
- 메타데이터를 왜 `export` 해야 봇이 읽나 (export가 뭔지도)
- 정적 메타데이터 vs 동적 메타데이터는 어떤 경우인가
- `generateMetadata` 함수명은 고정인가
- robots.txt와 sitemap.xml의 역할 차이
- OpenGraph로 이미지를 만든다는 게 뭔가

## 정리

### 메타데이터란
`<head>`의 `<title>`, `<meta>` 태그로 **사이트 정보를 넣어 봇(검색엔진)이 읽게** 하는 것(SEO). SSG/SSR이 완성 HTML을 봇에게 주는데, 그 head에 이 태그들이 있어야 봇이 읽는다.

**중요**: 봇은 내 `export`를 직접 읽는 게 아니라, 그걸로 **Next.js가 생성한 `<head>` HTML**을 읽는다.
```
export const metadata = {...}  →  Next.js가 <head><title>...</head> 생성  →  봇이 그 태그 읽음
```
- `export`는 "내보내다"(≠ import "불러오다"). 내가 만든 metadata를 밖으로 내놓아 Next.js가 가져다 쓰게 하는 것.
- App Router는 `metadata` export 방식, Pages Router는 `next/head`의 `<Head>` 컴포넌트 방식. 결과는 같음(head에 태그 넣기).

### 정적 vs 동적 메타데이터
기준: "title/description을 미리 알 수 있나?"

| | 정적 | 동적 |
|---|---|---|
| 문법 | `export const metadata = {...}` | `export async function generateMetadata()` |
| 값 | 고정(미리 정함) | 데이터/파라미터 따라 만듦 |
| 쓰는 곳 | 고정 페이지(소개·홈·약관) | 동적 경로(`[slug]`·`[id]` — 글·상품) |

```jsx
// 정적
export const metadata = { title: '회사 소개', description: '...' };

// 동적
export async function generateMetadata({ params }) {
  const post = await getPost(params.slug);
  return { title: post.title, description: post.summary };
}
```

- 동적 경로(`[]`)가 나오면 동적 메타데이터가 따라온다. 글마다 title이 달라야 검색 결과에서 구분됨(모든 글이 같은 title이면 SEO에 치명적).
- **`metadata`(정적), `generateMetadata`(동적) 둘 다 이름 고정.** `page.js` 같은 파일 이름 규칙처럼 export 이름도 Next.js와의 약속이라, 바꾸면 인식 안 됨. `generateMetadata`는 `params`로 동적 경로 값을 받는다.

### robots.txt — 접근 규칙
봇한테 **어디를 크롤링해도 되고 안 되는지 규칙**을 적는 파일(사이트 루트).
```
User-agent: *
Disallow: /admin/        # 여긴 긁지 마
Allow: /
Sitemap: https://내사이트.com/sitemap.xml   # 사이트맵 위치 안내
```
페이지 목록이 아니라 허용/차단 규칙 + 사이트맵 위치.

### sitemap.xml — 경로 목록
내 사이트에 **어떤 페이지들이 있는지** URL을 나열(봇이 다 찾아 색인하게).
```xml
<url><loc>https://내사이트.com/blog/react</loc><lastmod>2026-07-21</lastmod></url>
```
- robots.txt = 규칙 / sitemap.xml = 페이지 목록. robots가 사이트맵 위치를 가리켜 둘이 세트로 동작.
- 비유: robots = 출입 규칙 안내판, sitemap = 층별 안내도.
- Next.js: `app/robots.ts` → `/robots.txt`, `app/sitemap.ts` → `/sitemap.xml` 자동 생성(또 약속된 이름).

### OpenGraph / og:image — 링크 공유 미리보기
카톡·슬랙·트위터에 링크 붙였을 때 뜨는 **미리보기 카드**(제목·설명·썸네일)를 정하는 meta 태그들(Open Graph Protocol).
```html
<meta property="og:title" content="...">
<meta property="og:description" content="...">
<meta property="og:image" content="https://.../og.png">   <!-- 썸네일 -->
```
- 메타데이터의 일부 — `metadata` export의 `openGraph`에 넣음.
- `og:image` = 카드의 썸네일. 없으면 링크가 밋밋하게 뜸.
- **"이미지를 만든다"** = Next.js가 `opengraph-image.tsx`에서 **JSX를 PNG로 자동 생성**(`next/og`의 `ImageResponse`). 직접 디자인 안 해도 됨.
  ```tsx
  // app/blog/[slug]/opengraph-image.tsx
  import { ImageResponse } from 'next/og';
  export default function Image({ params }) {
    return new ImageResponse(<div style={{...}}>{params.slug}</div>);
  }
  ```
- `[slug]` 동적 경로와 결합하면 **글마다 제목 박힌 썸네일이 자동 생성**(손으로 이미지 안 만들어도 됨).

## 아직 헷갈리는 것 / 다음에 볼 것
- `opengraph-image.tsx`에서 폰트·배경 이미지 넣는 실제 코드.
- robots의 `Disallow`와 `noindex` 메타의 차이(크롤링 vs 색인).

## 한 줄 요약
메타데이터는 `<head>`에 사이트 정보를 넣어 봇이 읽게 하는 것(정적 `metadata` / 동적 `generateMetadata`, 이름 고정), robots.txt는 크롤링 규칙, sitemap.xml은 페이지 목록, OpenGraph는 링크 공유 미리보기 카드(og:image는 그 썸네일, Next.js는 `opengraph-image`로 코드 생성 가능) — 전부 head의 meta이거나 약속된 이름의 파일로 처리한다.
