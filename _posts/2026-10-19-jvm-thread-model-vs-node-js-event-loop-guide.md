---
title: "JVM 스레드 모델과 Node.js 이벤트 루프 심층 비교 가이드"
date: 2026-10-19 22:15:00 +0900
categories: [공부, 아키텍처]
tags: [jvm, springboot, nodejs, event-loop, thread-model]
---

이 문서는 **Tomcat 스레드 풀(멀티스레드 블로킹 모델)**과 **Node.js(싱글 스레드 이벤트 루프 모델)**의 아키텍처 차이를 개념, 실행 흐름(Call Stack, Task Queue, Microtask Queue), 그리고 커널의 역할 관점에서 총정리한 학습 산출물입니다.

---

## 1. 핵심 아키텍처 비교: 스프링 부트(Tomcat) vs Node.js

| 구분 | 스프링 부트 (Tomcat 기반) | Node.js |
| :--- | :--- | :--- |
| **기본 모델** | 멀티스레드 (Thread-per-request, 요청당 1스레드) | 싱글 스레드 이벤트 루프 (Single-threaded Event Loop) |
| **I/O 처리 방식** | **블로킹(Blocking) I/O:** 스레드가 DB 조회나 API 호출 결과를 기다리며 멈춰 있음 | **비동기 Non-blocking I/O:** 요청을 OS 커널이나 libuv 스레드 풀에 던지고 즉시 다음 코드로 넘어감 |
| **자원 관리** | 스레드 풀(예: 200개)을 미리 만들어 두고, OS 커널 스케줄러에 스레드 CPU 할당을 위임 | 메인 스레드 1개로 코드를 실행하며, 백그라운드 I/O는 커널/스레드 풀에 위임 후 큐를 통해 순차 처리 |

---

## 2. 주요 개념 정리

### ① Call Stack (콜 스택)
* 자바스크립트 엔진이 현재 실행 중인 함수와 코드의 실행 순서를 기록하는 스택 자료구조입니다.
* 싱글 스레드인 Node.js에서 코드가 실제로 실행되는 공간입니다.

### ② Task Queue (Task Queue / Macrotask Queue)
* `setTimeout`, `setInterval`, 그리고 `poll` 단계의 I/O 콜백 등 비동기 작업이 끝난 뒤 실행되어야 할 콜백 함수들이 대기하는 큐입니다.
* 이벤트 루프의 각 페이즈별로 전용 큐(`timers`, `poll`, `check` 등)가 존재합니다.

### ③ Microtask Queue (마이크로태스크 큐)
* `Promise.then()`, `queueMicrotask()`, `process.nextTick()` 등 **우선순위가 매우 높은 비동기 작업**이 쌓이는 큐입니다.
* Call Stack이 텅 비어 있는 경우, 이벤트 루프가 다음 페이즈로 넘어가기 전에 **마이크로태스크 큐를 싹 다 비우는 작업이 최우선**으로 수행됩니다.

---

## 3. 요청 하나가 처리되는 과정 비교 (시각화)

### [A] 스프링 부트 (Tomcat): 멀티스레드 블로킹 흐름
1. **요청 접수:** 클라이언트 요청이 들어오면 Tomcat 스레드 풀에서 **빈 스레드 1개**를 통째로 할당합니다.
2. **비즈니스 로직 및 DB 조회:** 스레드가 코드를 실행하다가 DB 조회 코드를 만나면, **결과가 올 때까지 해당 스레드는 멈춘 상태(Blocking)**로 대기합니다.
3. **응답 반환:** DB 응답이 오면 스레드가 깨어나서 결과를 가공한 뒤 클라이언트에 HTTP 응답을 보냅니다.
4. *한계:* 동시 요청이 스레드 풀 개수(예: 200개)를 넘어가면, 201번째 요청부터는 스레드를 배정받지 못하고 큐에서 대기하거나 거절당합니다.

### [B] Node.js: 싱글 스레드 이벤트 루프 흐름
1. **요청 접수:** 메인 싱글 스레드가 요청을 받아 Call Stack에 올려 실행합니다.
2. **비동기 위임:** DB 조회나 파일 읽기 등 I/O 작업을 만나면, Node.js는 이를 직접 처리하지 않고 **OS 커널이나 libuv 스레드 풀**로 던져버립니다. (메인 스레드는 멈추지 않고 즉시 다음 코드로 이동 - Non-blocking)
3. **백그라운드 처리 (커널):** 최신 커널의 멀티스레딩/비동기 메커니즘이 백그라운드에서 작업을 처리합니다.
4. **큐 적재:** 작업이 끝나면 콜백 함수가 이벤트 루프의 큐(`poll` 큐 등)에 적재됩니다.
5. **이벤트 루프 실행:** Call Stack이 비어있을 때, 이벤트 루프가 큐에서 콜백을 꺼내어 Call Stack에 올려 실행합니다.

---

## 4. 실습 및 실행 순서 예측 (`Promise` vs `setTimeout` vs `nextTick`)

코드 실행 시 Call Stack, Microtask Queue, Task Queue(Event Loop)가 상호작용하는 순서의 예시입니다.

```javascript
console.log('1. 동기 코드 시작');

setTimeout(() => {
    console.log('4. setTimeout (Macrotask)');
}, 0);

Promise.resolve().then(() => {
    console.log('3. Promise.then (Microtask)');
});

process.nextTick(() => {
    console.log('2. process.nextTick (Highest Priority)');
});

console.log('5. 동기 코드 끝');
```

**출력 결과 및 이유:**
1. **`1. 동기 코드 시작` / `5. 동기 코드 끝`**: Call Stack에서 동기 코드가 먼저 실행됩니다.
2. **`2. process.nextTick`**: 현재 스코프(Call Stack)가 끝나자마자 이벤트 루프 페이즈로 넘어가기 전 **가장 먼저 새치기**하여 실행됩니다.
3. **`3. Promise.then`**: 마이크로태스크 큐에 있던 작업이 다음으로 처리됩니다.
4. **`4. setTimeout`**: 이벤트 루프의 `timers` 단계로 넘어가야 실행되므로 가장 나중에 처리됩니다.

---

## 5. 오늘의 완료 질문 정답

> **Q. Node.js의 싱글 스레드는 정확히 어느 실행 경로를 뜻하는가?**
>
> **A:** 자바스크립트 엔진이 코드를 실행하고, 이벤트 루프를 순회하며 큐에 쌓인 콜백을 꺼내 처리하는 **메인 실행 주체(Main Execution Thread)**를 뜻합니다. 즉, 전체 시스템이나 OS 커널 수준이 아니라 **"자바스크립트 코드와 콜백을 실행하는 메인 주체 1개"**를 의미하며, 파일 I/O나 네트워크 같은 무거운 작업은 백그라운드(커널 및 libuv 스레드 풀)의 멀티스레드 환경에 위임됩니다.
