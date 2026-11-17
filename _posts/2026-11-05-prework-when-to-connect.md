---
title: "숨은 iframe preload는 기기 조건을 보고 켠다"
date: 2026-11-05
categories: [일일 회고]
description: "업로드 창을 빨리 열려고 켠 숨은 iframe preload는, 저사양에서 첫 화면을 무겁게 만들 수 있어 기기 조건을 보고 켜기로 했습니다."
---

빠른 경로를 만들려고 켠 미리 불러오기가, 느린 기기에서는 첫 화면을 무겁게 만들 수 있습니다. 업로드 창이 늦게 열려 숨은 iframe을 처음부터 불러 두면 여는 순간을 줄일 수 있다고 봤고, 빠른 기기에서는 그 방법이 맞아 보였습니다. 다만 사양이 낮은 기기에서는 같은 방식이 페이지를 불러오는 동안 렉을 만들 수 있어, 그 기기에서는 미리 불러오지 않게 바꾸고 불필요한 스크립트도 빼 두었습니다.

이 글은 숨은 iframe으로 패널을 미리 불러오려 할 때, 저사양에서 첫 화면이 버벅일 수 있는지부터 가려야 하는 분께 도움이 됩니다.

## 한눈에 보는 결론

- display:none이어도 문서와 스크립트는 불러져 실행됩니다.
- 실험실에서 첫 페인트는 거의 그대로였고, 비용은 긴 작업으로 나타났습니다. 같은 분량의 일을 CPU 8배속에서 재면 긴 작업 총합 중앙값은 280ms에서 2,424ms로 늘었습니다.
- saveData, 메모리, 논리 프로세서 수를 보고, 값이 없거나 낮으면 켜지 않습니다.
- 저사양 기기에서는 미리 불러오지 않게 바꾸고, 불필요한 스크립트도 빼 두었습니다.

## 제약에 맞춰 고른 방법

목표는 업로드 창을 여는 기다림을 줄이되, 느린 기기에서 첫 화면을 무겁게 만들지 않는 것입니다. 항상 켜 두면 빠른 기기에서는 이득이지만, 같은 최적화가 느린 기기에서는 반대로 작동할 수 있어 버렸습니다.

기기 메모리 값만 보는 방법도 두지 않았습니다. navigator.deviceMemory는 대략적인 기비바이트를 2의 거듭제곱으로 거칠게 돌려주고, 보안 맥락에서만 쓸 수 있습니다. W3C는 2026년 10월 이 API를 Discontinued Draft로 두었고, 지원하지 않는 브라우저에서는 값이 없습니다. 없는 값을 고사양으로 보면 preload가 꺼지지 않습니다.

네트워크 상태만 보는 방법도 좁았습니다. navigator.connection.saveData는 사용자가 데이터 절약을 켠 신호라 존중할 만하지만, Network Information API 자체는 모든 브라우저의 기준 기능이 아니고, 느린 CPU를 직접 말해 주지는 않습니다. 그래서 여러 신호를 보되, 하나가 없거나 낮으면 켜지 않기로 했습니다.

## 기본 구조

iframe 문서는 중첩된 탐색 공간입니다. display:none이어도 문서를 불러오고 스크립트를 실행하는 일은 멈추지 않습니다. 같은 출처이면 그 일이 부모 문서와 스레드를 나눠 쓰는 경우가 많아, 프레임 안의 긴 작업이 부모 화면의 입력을 밀 수 있습니다.

![숨은 iframe preload는 조건을 통과할 때만 켠다](/dev-articles/assets/2026-11-05-prework-when-to-connect-01.png)

켤 조건을 정할 때는 표준화된 값부터 봤습니다. navigator.hardwareConcurrency는 HTML에 정의된 논리 프로세서 수이고, 브라우저가 더 작은 수를 돌려줄 수 있습니다. saveData가 true이면 preload를 끕니다. deviceMemory가 있고 그 값이 낮으면 끄고, 값이 없으면 고사양으로 보지 않습니다.

```js
function shouldPreload({ connection, deviceMemory, hardwareConcurrency } = navigator) {
  if (connection && connection.saveData) return false
  if (typeof deviceMemory === 'number' && deviceMemory < 4) return false
  if (typeof hardwareConcurrency === 'number' && hardwareConcurrency <= 4) return false
  if (typeof deviceMemory !== 'number') return false
  return true
}

export function mountPreloadedPanel(doc, src) {
  if (!shouldPreload()) return null
  const frame = doc.createElement('iframe')
  frame.src = src
  frame.style.display = 'none'
  frame.title = '미리 불러온 패널'
  doc.body.appendChild(frame)
  return frame
}
```

## 미리 불러오기를 켠 뒤

### 처음 판단: 미리 불러오면 창이 빨리 열림

업로드 창을 빨리 열려고 숨은 iframe을 처음부터 불러 두었습니다. 빠른 기기에서는 기다림이 줄어 이 방법이 맞아 보였습니다.

### 저사양 기기에서 미리 불러오기 끄기

사양이 낮은 기기에서는 같은 방식이 페이지를 불러오는 동안 렉을 만들 수 있어, 그 기기에서는 미리 불러오지 않게 바꾸고 불필요한 스크립트도 빼 두었습니다. 그 렉이 어디서 생기는지는, 첫 페인트가 아니라 숨은 프레임이 만든 긴 작업 쪽이었습니다.

## 한계

deviceMemory는 비표준이고, 지금은 Discontinued Draft입니다. hardwareConcurrency는 HTML 기준 기능이지만 실제 코어 수보다 작게 보고될 수 있습니다. saveData는 브라우저에 따라 없을 수 있습니다. 신호를 보수적으로 보면 고사양에서도 가끔 꺼질 수 있고, preload를 끄면 빠른 기기에서는 창을 여는 순간 한 번 더 기다리게 됩니다. 그 비용을 감수하고, 기본값은 켜지 않는 쪽에 두었습니다.

## 확인 방법

확인은 페인트 시각과 긴 작업을 같이 봅니다. PerformancePaintTiming의 first-contentful-paint와, 긴 작업 항목의 개수와 합을 적습니다. CPU를 느리게 둔 뒤에도 같은 분량의 일이 길어지는지, shouldPreload()가 값이 없을 때 false를 주는지를 봅니다.

Headless Chrome 157에서 부모 문서에 제목을 바로 그려 두고, display:none iframe이 고정된 양의 일을 하는 스크립트를 불러오게 했습니다. 이 환경의 deviceMemory는 16, hardwareConcurrency는 8, saveData는 false였고, localhost라 보안 맥락은 true였습니다. preload가 없을 때 first-contentful-paint 중앙값은 1배속에서 40ms, 8배속에서는 124ms였고 긴 작업은 거의 없었습니다. 숨은 iframe을 붙이면 페인트는 1배속에서 40ms, 8배속에서 164ms로 비슷했지만, 긴 작업 총합 중앙값은 280ms에서 2,424ms로 늘었습니다.

저는 업로드 창을 빨리 열려고 항상 켜 두었다가, 사양이 낮은 기기에서 첫 화면이 무거워질 수 있어 그 기기에서는 끄기로 했습니다. 다시 재어 보니 그 비용은 페인트 숫자보다 긴 작업으로 나타났고, 같은 일이 느린 CPU에서 훨씬 길었습니다.

## 정리

- 숨은 프레임도 부모와 스레드를 나눕니다.
- 첫 페인트만 보면 비용을 놓칩니다.
- 신호가 없거나 낮으면 켜지 않습니다.

숨은 preload는 항상 켜 두지 않고, 기기와 네트워크가 감당할 때만 켜려고 합니다.

## 참고

- [MDN deviceMemory](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/deviceMemory)
- [W3C Device Memory API](https://www.w3.org/TR/device-memory/)
- [MDN hardwareConcurrency](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/hardwareConcurrency)
- [MDN Network Information](https://developer.mozilla.org/en-US/docs/Web/API/NetworkInformation)
- [MDN PerformancePaintTiming](https://developer.mozilla.org/en-US/docs/Web/API/PerformancePaintTiming)
