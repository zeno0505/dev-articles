---
title: "같은 오류 코드로는 중단과 시간 초과를 같이 끄지 않는다"
date: 2026-11-04
categories: [일일 회고]
description: "axios 0.21.4는 중단과 시간 초과를 ECONNABORTED로 같이 남겨, 브라우저가 끊은 요청만 보고에서 빼고 시간 초과는 남겼습니다."
---

한 오류 코드 아래를 통째로 끄면, 남겨야 할 신호까지 같이 사라집니다. 오류 알림이 크게 늘었을 때, 다른 앱에서 연 웹 화면이 다시 앱으로 넘어가며 끊긴 요청이 한꺼번에 잡힌 것으로 보였고, 그 실패 코드를 끄면 조용해질 거라고 생각했습니다. 다만 쓰던 클라이언트 axios 0.21.4는 브라우저가 끊은 요청과 시간 초과를 같은 오류 코드 ECONNABORTED로 남겨, 그 코드를 끄면 실제 시간 초과까지 안 보이게 됩니다. 그래서 브라우저가 끊은 요청만 보고에서 빼고, 시간 초과는 남겨 두었습니다.

이 글은 실패 알림이 늘었을 때 오류 코드를 묶어 끄려 하거나, 중단과 시간 초과가 같은 칸에 있는 분께 도움이 됩니다.

## 과제와 제약

목표는 예상된 중단이 알림을 채우지 않게 하되, 실제 시간 초과와 고쳐야 할 실패는 보고에 남기는 것입니다. 화면이 닫히거나 다른 앱으로 넘어가며 요청이 끊기는 일은 자주 있습니다. 그 중단만 끄고, 같은 코드 아래의 시간 초과는 끄지 않아야 했습니다. 예상된 업무 4xx를 상태 코드만으로 끄면, 같은 상태에 숨은 다른 원인도 놓칩니다.

## 한눈에 보는 결론

- axios 0.21.4는 중단과 시간 초과에 같은 ECONNABORTED를 씁니다. 그 코드를 통째로 끄면 둘 다 사라집니다.
- 브라우저가 끊은 요청만 보고에서 빼고, 시간 초과는 남겼습니다.
- 공식 fetch 경로에서는 중단이 AbortError, 시간 초과가 TimeoutError이고, 4xx는 fetch가 거부하지 않습니다.
- 예상된 업무 4xx를 알림에서 거른 뒤, 남은 실제 오류는 고치기 전과 후를 다시 재현해 고쳤습니다.

## 기본 구조

![같은 오류 코드 아래를 원인으로 나눈다](/dev-articles/assets/2026-11-04-mute-by-cause-01.png)

실제 고친 모양은, 공유 코드가 아니라 원인 이름으로 보고 여부를 가리는 함수입니다. 공식 경로의 AbortError는 끄고, TimeoutError는 남깁니다. axios의 ECONNABORTED는 메시지나 원인 필드로 시간 초과와 중단을 나눕니다.

```js
export function shouldReport(error, response) {
  if (error && error.name === 'AbortError') return false
  if (error && error.name === 'TimeoutError') return true
  if (error && error.code === 'ECONNABORTED') {
    const message = String(error.message || '')
    if (/timeout/i.test(message)) return true
    return false
  }
  if (response && response.status >= 400 && response.status < 500) {
    const cause = response.body && response.body.cause
    if (cause && cause.expected) return false
  }
  return true
}
```

같은 출처의 느린 응답을 두고, 한쪽은 AbortController.abort()로 끊고 다른 쪽은 AbortSignal.timeout(50)으로 끊으면, 예외 이름은 AbortError와 TimeoutError로 갈렸습니다. 둘 다 DOMException이었습니다. 400 응답은 fetch가 거부하지 않았고, 상태와 본문을 그대로 주었습니다. 이 둘을 REQUEST_FAILED라는 한 코드로 묶은 뒤 그 코드를 끄면, 중단과 시간 초과가 함께 사라집니다. 원인 이름 기준으로 중단과 예상된 400만 끄면, 보고에 남는 것은 TimeoutError뿐입니다. 공식 문서도 AbortSignal.timeout이 시간 초과 때 TimeoutError를 쓰고, 사용자의 중단은 AbortError로 나누어 보라고 적습니다.

## 알림을 끄기까지

### 처음 판단: 공유 코드를 통째로 끄면 조용해질 것

알림이 늘었을 때 가장 쉬운 대응은 그 실패 코드를 통째로 끄는 것이었습니다. 다만 같은 칸에 실제 시간 초과가 들어 있으면, 시끄러운 쪽만 끄려다 장애 신호까지 같이 꺼집니다.

### 브라우저가 끊은 요청만 제외, 시간 초과는 유지

쓰던 클라이언트가 두 원인을 같은 코드로 남긴다는 점을 본 뒤, 브라우저가 끊은 요청만 보고에서 빼기로 했습니다. 시간 초과는 그대로 두었습니다.

### 예상된 4xx 거르기와 남은 오류의 전후 재현

예상된 업무 응답이 알림을 가득 채우는 경우도 결이 같습니다. 정상적인 4xx를 상태 코드만으로 끄지 않고, 본문 원인까지 본 뒤 예상된 것만 거르고, 남은 실제 오류는 고치기 전과 후를 다시 재현해 고쳤습니다.

## 한계

원인 단위로 나누면 분류 규칙이 늘어납니다. fetch는 4xx만으로 약속을 거부하지 않으므로, 상태만 보고 끄면 같은 상태 아래의 다른 본문 원인까지 같이 사라질 수 있습니다. 메시지 문자열만 믿으면 클라이언트가 바뀌었을 때 구분이 흔들릴 수 있어, 공식 경로에서는 예외 이름을 먼저 봅니다.

## 확인 방법

실패한 요청을 세 갈래로 재현합니다. AbortController로 끊은 요청, AbortSignal.timeout으로 끊은 요청, 그리고 예상된 4xx입니다. 각각에서 error.name, error.message, HTTP 상태를 적고, 한 코드로 끈 뒤의 남은 알림과 원인 단위로 끈 뒤의 남은 알림을 비교합니다. 시간 초과는 후자에만 남아야 합니다.

저는 알림이 늘었을 때 공유 코드를 통째로 끄면 편할 것처럼 보였습니다. 다시 보니 그 코드 아래에는 실행이 멈춘 중단과, 기다리다 실패한 시간 초과가 함께 있었습니다. 브라우저가 끊은 요청만 빼자 시간 초과는 보고에 남았고, 예상된 4xx를 거른 뒤에는 남은 실패를 고치기 전과 후로 재현할 수 있었습니다.

## 정리

- 같은 오류 코드 아래를 통째로 끄지 않습니다.
- 공식 경로에서는 AbortError와 TimeoutError를 나눕니다.
- 예상된 4xx도 상태만이 아니라 본문 원인을 같이 봅니다.

실패 알림은 코드 하나가 아니라, 그 아래에 있는 원인부터 보고 끄려고 합니다.

## 참고

- [MDN fetch 예외](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch#exceptions)
- [MDN AbortSignal.timeout](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static)
