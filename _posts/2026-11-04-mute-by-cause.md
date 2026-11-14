---
title: "같은 오류 코드로는 중단과 시간 초과를 같이 끄지 않는다"
date: 2026-11-04
categories: [일일 회고]
---

요청이 실패했다는 이유만으로 알림을 한 코드로 묶으면, 일부러 중단한 요청과 시간이 초과된 요청이 같이 꺼질 수 있습니다. fetch를 AbortController로 중단하면 AbortError가 나고, AbortSignal.timeout으로 끊으면 TimeoutError가 납니다. 이 둘을 REQUEST_FAILED 같은 한 칸에 넣으면, 시간을 넘겨 실패한 신호까지 보고에서 사라집니다. 그래서 알림은 코드가 아니라 원인 단위로 끄기로 했습니다.

## 한 코드로 묶이면 남은 실패가 보이지 않습니다

화면이 닫히거나 다른 앱으로 넘어가며 요청을 끊는 일은 자주 있습니다. 그 중단이 알림을 크게 늘리면, 가장 쉬운 대응은 그 실패 코드를 통째로 끄는 것입니다. 다만 같은 칸에 실제 시간 초과가 들어 있으면, 시끄러운 쪽만 끄려다 장애 신호까지 같이 꺼집니다. 예상된 업무 응답이 알림을 가득 채우는 경우도 결이 같습니다. 실패로 보이지만 정상적인 4xx를 상태 코드만으로 끄면, 같은 상태에 숨은 다른 원인을 놓칩니다.

## 중단과 시간 초과는 예외 이름이 다릅니다

같은 출처의 느린 응답을 두고, 한쪽은 AbortController.abort()로 끊고 다른 쪽은 AbortSignal.timeout(50)으로 끊었습니다. 예상된 업무 실패는 400과 함께 본문에 원인 코드를 담아 내려 보냈습니다.

```js
const controller = new AbortController()
const aborted = fetch('/slow', { signal: controller.signal })
  .catch((error) => error)
controller.abort()

const timedOut = fetch('/slow', { signal: AbortSignal.timeout(50) })
  .catch((error) => error)

const business = await fetch('/biz')
```

중단된 요청의 예외 이름은 AbortError였고, 메시지는 This operation was aborted였습니다. 시간 초과의 이름은 TimeoutError였고, 메시지는 The operation was aborted due to timeout이었습니다. 둘 다 DOMException이었습니다. 400 응답은 fetch가 거부하지 않았고, 상태와 본문을 그대로 주었습니다.

이 둘을 REQUEST_FAILED라는 한 코드로 묶은 뒤 그 코드를 끄면, 중단과 시간 초과가 함께 사라집니다. 원인 이름 기준으로 중단과 예상된 400만 끄면, 보고에 남는 것은 TimeoutError뿐입니다. 공식 문서도 AbortSignal.timeout이 시간 초과 때 TimeoutError를 쓰고, 사용자의 중단은 AbortError로 나누어 보라고 적습니다.

## 끄는 단위는 예외 이름과 상태의 의미를 같이 봅니다

확인할 때는 실패한 요청을 세 갈래로 재현합니다. AbortController로 끊은 요청, AbortSignal.timeout으로 끊은 요청, 그리고 예상된 4xx입니다. 각각에서 error.name, error.message, HTTP 상태를 적고, 한 코드로 끈 뒤의 남은 알림과 원인 단위로 끈 뒤의 남은 알림을 비교합니다. 시간 초과는 후자에만 남아야 합니다.

fetch는 4xx만으로 약속을 거부하지 않습니다. 상태 코드를 한 번에 끄면 그 상태 아래의 다른 본문 원인까지 같이 사라질 수 있어, 본문 원인과 예외 이름을 같이 봅니다.

원인 단위로 나누면 분류 규칙이 늘어납니다. 그 대신 시간 초과 같은 실제 장애 신호를 끄지 않을 수 있습니다. 한 코드로 빠르게 조용히 만드는 쪽은, 이번 재현에서 시간 초과까지 같이 지워 버려서 택하지 않았습니다.

예전에 알림이 늘었을 때 코드를 통째로 끄면 편할 것처럼 보였습니다. 다시 재어 보니 그 코드 아래에는 실행이 멈춘 중단과, 기다리다 실패한 시간 초과가 함께 있었습니다. 예상된 4xx를 걸러 내는 일도 같은 이유입니다. 남은 실패는 고치기 전과 후를 재현할 수 있게, 보고에 남겨 두려고 합니다. 보고 경로를 넓힐 때 어떤 값이 함께 나가는지는 이 글에서 다루지 않습니다.

실패 알림은 코드 하나가 아니라, 그 아래에 있는 원인부터 보고 끄려고 합니다.
