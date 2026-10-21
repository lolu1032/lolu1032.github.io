---
title: "스프링 부트 개발자가 본 NestJS 입문기: 개념부터 라우팅, 요청 처리까지"
date: 2026-10-21 22:00:00 +0900
categories: [공부, NestJS]
tags: [nestjs, spring-boot, backend, typescript, routing, di]
---

안녕하세요! 스프링 부트(Spring Boot)에 익숙한 개발자가 자바스크립트/타입스크립트 진영의 NestJS를 처음 접했을 때 느낀 차이점과 핵심 개념을 총정리해 보았습니다. 스프링 부트와 1:1로 비교하며 이해하면 NestJS의 구조가 훨씬 쉽게 다가옵니다.

---

## 1. 아키텍처의 뼈대: "껍데기만 바뀐 스프링?"

NestJS를 처음 보면 데코레이터(`@`)가 많아 번거로워 보이지만, 스프링 부트의 철학(DI, IoC, 모듈화)을 TypeScript 생태계에 거의 그대로 가져온 구조입니다.

| 개념 | Spring Boot | NestJS |
| :--- | :--- | :--- |
| **컨트롤러** | `@RestController` | `@Controller` |
| **서비스** | `@Service` | `@Injectable` |
| **의존성 주입** | 생성자 주입 (`private final Service`) | 생성자 주입 (`constructor(private service: Service)`) |
| **모듈 설정** | `@Configuration` / `@Module` | `@Module` |

실제 서비스 계층을 연결하는 코드 형태도 스프링 부트와 완벽히 일치합니다.

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  findAll(): string {
    return 'This action returns all cats from service!';
  }
}
```

```typescript
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
export class CatsController {
  // 스프링의 생성자 주입과 정확히 같은 개념!
  constructor(private readonly catsService: CatsService) {}

  @Get()
  findAll(): string {
    return this.catsService.findAll();
  }
}
```

---

## 2. 요청 파라미터 매핑 비교 (@Req, @Body, @Param, @Query)

스프링 부트에서 쓰던 요청 처리 어노테이션들은 NestJS에서도 이름과 역할이 거의 똑같습니다.

* `@Req()` $\rightarrow$ 스프링의 `HttpServletRequest` (요청 객체 직접 접근, 헤더/쿠키/IP 추출)
* `@Body()` $\rightarrow$ 스프링의 `@RequestBody` (요청 본문(JSON)을 DTO 클래스로 바인딩)
* `@Param('id')` $\rightarrow$ 스프링의 `@PathVariable("id")` (URL 경로 변수 추출)
* `@Query()` $\rightarrow$ 스프링의 `@RequestParam` 또는 쿼리 파라미터 바인딩 (`?limit=10`)

---

## 3. 라우팅 충돌 이슈와 해결 순서 (Express 기반의 한계)

NestJS는 기본적으로 Express 엔진 위에서 동작하므로, 코드가 작성된 위에서 아래로 선형 검색(Linear Search)을 통해 라우트를 등록하고 매칭합니다.

### ⚠️ 주의해야 할 라우트 충돌

```typescript
@Controller('users')
export class UsersController {
  @Get(':id')   // 만능 주머니(:id)가 위에 있으면...
  findOne() {}

  @Get('me')    // 절대 도달할 수 없음 (never reached)
  findMe() {}
}
```

`/users/me`로 요청을 보내더라도 첫 번째 `@Get(':id')`가 `me`를 파라미터로 먼저 가로채기 때문에 `findMe()`는 실행되지 않습니다.

### ✅ 해결 방법

> **"구체적이고 특정한 주소는 위에, 아무거나 다 받아먹는 파라미터 주소(`:id`)는 아래에!"**

```typescript
@Controller('users')
export class UsersController {
  @Get('me')    // 정적이고 구체적인 경로를 먼저 선언!
  findMe() {}

  @Get(':id')   // 범용적인 파라미터 경로는 아래로 내리기
  findOne() {}
}
```

---

## 4. 응답 상태 코드 제어 (@HttpCode vs @Res)

* **고정 상태 코드**: `@HttpCode(204)` 데코레이터를 메서드 위에 붙여서 간편하게 지정 (스프링의 `@ResponseStatus`와 동일)
* **동적 제어**: 스프링의 `HttpServletResponse`처럼 로우단 제어가 필요할 때는 `@Res({ passthrough: true }) res: Response`를 사용해 `res.status()`를 조작 가능 (가급적 NestJS의 기본 자동 응답 기능을 쓰는 것이 권장됨)

---

## 5. 💡 공식 문서 예제의 함정: 더미(Test) 문자열

공식 문서나 튜토리얼을 보면 컨트롤러에서 아래와 같은 코드를 흔히 볼 수 있습니다.

```typescript
@Get()
findAll(@Query() query: ListAllEntities) {
  return `This action returns all cats (limit: ${query.limit} items)`;
}
```

* **이 문구의 정체**: 시스템 키워드가 아니라, 요청이 정상적으로 들어오고 매핑되는지 확인하기 위해 공식 문서가 임의로 적어둔 테스트용(더미) 문자열입니다.
* **실무에서는**: 저 자리에 문자열을 직접 리턴하지 않고, DB에서 조회한 진짜 데이터(JSON 객체/배열)나 공통 응답 포맷을 리턴하게 됩니다!

---

## 총평

스프링 부트를 다뤄본 개발자라면 NestJS의 구조가 처음엔 "데코레이터가 너무 많다"고 느껴질 수 있지만, DI와 모듈 중심의 아키텍처 철학이 거의 일치하기 때문에 적응하기 매우 수월합니다. 라우팅 순서와 데코레이터 특징만 잘 기억하면 타입스크립트 기반의 강력한 백엔드 서버를 빠르게 구축할 수 있습니다!
