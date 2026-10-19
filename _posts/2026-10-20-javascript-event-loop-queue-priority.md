---
title: "[JS/Node] 이벤트 루프와 큐 우선순위 완벽 정리: 왜 내 setTimeout은 제시간에 안 돌까?"
date: 2026-10-20 01:00:00 +0900
categories: [공부, JavaScript]
tags: [javascript, nodejs, event-loop, call-stack, microtask, macrotask]
---

자바스크립트의 비동기 동작 원리를 공부하다 보면 Call Stack, Task Queue, Microtask Queue의 관계에서 혼란을 겪기 쉽습니다. 특히 **"타이머 시간이 다 끝났는데 왜 내 콜백은 바로 실행되지 않을까?"**라는 의문은 이벤트 루프의 우선순위 규칙을 이해하면 명쾌하게 풀립니다.

실무 관점에서 핵심 개념과 동작 메커니즘을 깔끔하게 정리했습니다.

---

## 1. 런타임의 3대 핵심 컴포넌트

자바스크립트 엔진 자체는 단 하나의 Call Stack을 사용하는 싱글 스레드입니다. 한 번에 한 가지 일만 할 수 있기 때문에, 비동기 처리는 브라우저(Web APIs)나 Node.js(libuv + OS 커널)라는 외부 백그라운드 환경과의 협업으로 이루어집니다.

### ① Call Stack (호출 스택 - 동기 코드)
* 현재 실행 중인 함수 프레임들이 쌓였다가(push) 완료되면 제거되는(pop) 단 하나의 실행 통로입니다.
* `console.log`, 사칙연산, 변수 선언 등 일반적인 동기 코드가 이곳에서 즉시 실행됩니다.
* **주의**: '마이크로스택'이라는 개념은 없으며, 스택은 Call Stack 단 하나뿐입니다.

### ② Microtask Queue (마이크로태스크 큐 - 최우선 비동기)
* `Promise.then()`, `.catch()`, `.finally()`, `async/await` 뒷부분, `queueMicrotask()` 전용 큐입니다.
* Node.js 환경의 `process.nextTick()` 역시 마이크로태스크 성격(심지어 Promise보다 한 단계 더 높은 최우선 새치기)을 띱니다.

### ③ Task Queue / Macrotask Queue (태스크 큐 - 일반 비동기)
* `setTimeout`, `setInterval`, I/O 콜백(`setImmediate` 포함) 등 전통적인 비동기 콜백들이 대기하는 큐입니다.

---

## 2. 이벤트 루프의 절대적인 실행 우선순위 규칙

이벤트 루프는 Call Stack이 비어 있을 때 다음의 엄격한 규칙에 따라 작업을 스택으로 올려 보냅니다.

```text
[ 동기 코드 실행 (Call Stack) ]
             ↓
     [ Call Stack이 텅 빔 ]
             ↓
   [ Microtask Queue 확인 ] ──(작업 있음)──> [ 큐가 완전히 바닥날 때까지 전부 실행 ]
             ↓ (완전히 비워짐)
[ Task Queue에서 작업 딱 '1개' 꺼내서 실행 ] ──> (1개 끝날 때마다 다시 Microtask 확인)
```

1. **1순위 (동기)**: Call Stack에 있는 동기 코드가 무조건 최우선으로 전부 실행됩니다.
2. **2순위 (Microtask)**: 동기 코드가 끝나 스택이 비는 즉시 Microtask Queue로 달려갑니다. 이 큐에 들어있는 작업은 큐가 완전히 빌 때까지(Empty) 계속해서 처리됩니다.
3. **3순위 (Task)**: Microtask Queue가 완전히 100% 비워진 뒤에야 비로소 Task Queue에서 단 1개의 태스크를 꺼내 실행합니다. 태스크를 하나 실행하고 나면, 다음 태스크로 가기 전에 또다시 Microtask Queue가 비어있는지 확인합니다.

---

## 3. 질문: "타이머가 다 됐는데 Microtask가 차면 어떻게 될까?"

> **결론**: 무조건 밀립니다. (Starvation 현상)

`setTimeout(fn, 1000)`에서 1000ms는 **"1초 뒤에 정확히 실행된다"**는 뜻이 아닙니다. **"1초 뒤에 Task Queue에 들어갈 자격을 얻는다"**는 의미입니다.

타이머 시간이 다 되어 이미 Task Queue 맨 앞에서 대기 중이더라도, Call Stack이 돌고 있거나 Microtask Queue에 작업이 남아있다면 `setTimeout` 콜백은 절대로 실행될 수 없습니다.

### 🧪 실험 코드로 확인하기

```javascript
// 0초 뒤 실행 예약 -> 사실상 즉시 Task Queue 맨 앞에 등록됨
setTimeout(() => {
  console.log('3. setTimeout 실행! (한참 기다림)');
}, 0);

// Microtask Queue에 등록 & 꼬리물기
Promise.resolve()
  .then(() => {
    console.log('1. Promise 1차 완료');
    return Promise.resolve(); // 새로운 Microtask 추가
  })
  .then(() => {
    console.log('2. Promise 2차 완료');
  });

console.log('0. 동기 코드 실행');
```

### 💻 실행 결과

```text
0. 동기 코드 실행
1. Promise 1차 완료
2. Promise 2차 완료
3. setTimeout 실행! (한참 기다림)
```

1. 동기 코드 `console.log('0...')`가 먼저 실행됩니다.
2. Call Stack이 비자 이벤트 루프는 Task Queue에 이미 도착해 있는 `setTimeout`을 무시하고, Microtask Queue를 먼저 텁니다.
3. 중간에 Promise 체이닝으로 새로운 마이크로태스크가 들어오자, 이벤트 루프는 그 새로운 작업까지 끝까지 다 비운 뒤에야 비로소 Task Queue로 내려가 `setTimeout`을 실행합니다.

---

## 4. 백그라운드 API를 지도 서비스에 빗대어 이해하기

왜 이런 복잡한 구조를 쓸까요? 자바스크립트는 싱글 스레드이기 때문입니다.

* **메인 스레드 (내 코드)**: 손님 응대하는 카운터 직원 1명입니다.
* **백그라운드 (Web API / OS 커널)**: 카카오맵, 다음지도 API처럼 "외부 전문 외주 엔진"입니다.

내가 직접 무겁게 지도 위성 좌표를 계산하고 타일을 그리는 대신, 외부 지도 API에 "결과 나오면 알려줘" 하고 던져둡니다. 그동안 직원은 멈추지 않고 다른 손님(다른 요청)을 받습니다.

외부 엔진에서 계산이 끝나면 결과 콜백을 큐에 전달하고, 직원은 한가할 때(스택이 비었을 때) 큐에서 결과를 꺼내 화면에 띄우기만 합니다.

---

## 5. 핵심 요약 (Cheat Sheet)

* **스택은 하나**: Call Stack 하나만 존재하며, Microtask는 스택이 아니라 큐(Queue)입니다.
* **우선순위 철칙**: 동기 코드(Call Stack) > Microtask Queue(Promise, nextTick) > Task Queue(setTimeout, I/O)
* **지연의 이유**: `setTimeout` 지연 시간은 "실행 시간"이 아니라 "Task Queue 진입 시간"입니다. 앞선 마이크로태스크나 동기 코드가 길어지면 실제 실행은 얼마든지 뒤로 밀릴 수 있습니다.
