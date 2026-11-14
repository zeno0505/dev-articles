---
title: "숨은 iframe preload는 기기 조건을 보고 켠다"
date: 2026-11-05
categories: [일일 회고]
---

업로드 창이 늦게 열려, 숨은 iframe을 처음부터 불러 두면 여는 순간을 줄일 수 있다고 보고 미리 로드를 넣었습니다. 빠른 기기에서는 기다림이 줄어 이 길이 맞아 보였습니다. 다만 사양이 낮은 기기에서는 같은 방식이 페이지를 불러오는 동안 렉을 만들 수 있어, 그 기기에서는 미리 불러오지 않게 바꾸고 불필요한 스크립트도 빼 두었습니다. 그 렉이 어디서 생기는지는, 첫 페인트가 아니라 숨은 프레임이 만든 긴 작업 쪽이었습니다.

## 항상 켜 두거나, 한 신호만 보는 길은 버렸습니다

항상 켜 두는 쪽은 버렸습니다. 같은 최적화가 느린 기기에서는 첫 화면을 무겁게 만들 수 있었습니다.

기기 메모리 값만 보는 길도 두지 않았습니다. navigator.deviceMemory는 대략적인 기비바이트를 2의 거듭제곱으로 거칠게 돌려주고, 보안 맥락에서만 쓸 수 있습니다. W3C는 2026년 10월 이 API를 Discontinued Draft로 두었고, 지원하지 않는 브라우저에서는 값이 없습니다. 없는 값을 고사양으로 보면 preload가 꺼지지 않습니다.

네트워크 상태만 보는 길도 좁았습니다. navigator.connection.saveData는 사용자가 데이터 절약을 켠 신호라 존중할 만하지만, Network Information API 자체는 모든 브라우저의 기준 기능이 아니고, 느린 CPU를 직접 말해 주지는 않습니다. 그래서 여러 신호를 보되, 하나가 없거나 낮으면 켜지 않기로 했습니다.

## 숨은 프레임도 부모와 스레드를 나눕니다

iframe 문서는 중첩된 탐색 공간입니다. display:none이어도 문서를 불러오고 스크립트를 실행하는 일은 멈추지 않습니다. 같은 출처이면 그 일이 부모 문서와 스레드를 나눠 쓰는 경우가 많아, 프레임 안의 긴 작업이 부모 화면의 입력을 밀 수 있습니다.

켤 조건을 정할 때는 표준화된 값부터 봤습니다. navigator.hardwareConcurrency는 HTML에 정의된 논리 프로세서 수이고, 브라우저가 동시에 돌릴 수 있는 작업 수에 가깝게 더 작은 수를 돌려줄 수 있습니다. saveData가 true이면 사용자가 덜 쓰겠다고 한 것이므로 preload를 끕니다. deviceMemory가 있고 그 값이 낮으면 끄고, 값이 없으면 고사양으로 보지 않습니다.

```js
function shouldPreload() {
  if (navigator.connection && navigator.connection.saveData) return false
  const memory = navigator.deviceMemory
  const cores = navigator.hardwareConcurrency
  if (typeof memory === 'number' && memory < 4) return false
  if (typeof cores === 'number' && cores <= 4) return false
  if (typeof memory !== 'number') return false
  return true
}

if (shouldPreload()) {
  const frame = document.createElement('iframe')
  frame.src = '/upload-panel.html'
  frame.style.display = 'none'
  frame.title = '미리 불러온 패널'
  document.body.appendChild(frame)
}
```

## 첫 페인트만 보면 비용을 놓칩니다

Headless Chrome 157에서 부모 문서에 제목을 바로 그려 두고, display:none iframe이 고정된 양의 일을 하는 스크립트를 불러오게 했습니다. 이 환경의 deviceMemory는 16, hardwareConcurrency는 8, saveData는 false였습니다. localhost라 보안 맥락은 true였습니다.

preload가 없을 때 first-contentful-paint 중앙값은 1배속에서 40ms, 8배속에서는 124ms였고 긴 작업은 거의 없었습니다. 숨은 iframe을 붙이면 페인트는 1배속에서 40ms, 8배속에서 164ms로 비슷했지만, 긴 작업 총합 중앙값은 280ms에서 2,424ms로 늘었습니다. 화면의 첫 글자는 비슷하게 나와도, 느린 CPU에서는 그 뒤에 스레드가 오래 바빴습니다. 이 수치는 실험실에서 잰 값이고, 제품 체감을 대체하지는 않습니다.

확인은 페인트 시각과 긴 작업을 같이 봅니다. PerformancePaintTiming의 first-contentful-paint와, 긴 작업 항목의 개수와 합을 적습니다. CPU를 느리게 둔 뒤에도 같은 분량의 일이 길어지는지, shouldPreload()가 값이 없을 때 false를 주는지를 봅니다.

preload를 끄면 빠른 기기에서는 창을 여는 순간 한 번 더 기다리게 됩니다. 그 대신 느린 기기에서 첫 화면이 그려진 뒤 스레드를 오래 차지하지 않습니다. 신호를 보수적으로 보면 고사양에서도 가끔 꺼질 수 있습니다. 그 비용을 감수하고, 기본값은 켜지 않는 쪽에 두었습니다.

숨은 프레임을 미리 불러 두는 일은 빠른 경로를 만드는 최적화입니다. 저는 업로드 창을 빨리 열려고 항상 켜 두었다가, 사양이 낮은 기기에서 첫 화면이 무거워질 수 있어 그 기기에서는 끄기로 했습니다. 다시 재어 보니 그 비용은 페인트 숫자보다 긴 작업으로 나타났고, 같은 일이 느린 CPU에서 훨씬 길었습니다. 그래서 조건을 통과할 때만 켜기로 했습니다. 같은 날 미리 옮겨 둔 코드를 라우트와 끊어서 들고 가는 이야기는 이 글에서 다루지 않습니다.

숨은 preload는 항상 켜 두지 않고, 기기와 네트워크가 감당할 때만 켜려고 합니다.
