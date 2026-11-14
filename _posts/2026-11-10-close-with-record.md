---
title: "부모 창에 건 리스너는 pagehide에서 푼다"
date: 2026-11-10
categories: [일일 회고]
---

성능을 보려고 iframe 안의 앱이 부모 창에 긴 작업 관측기를 걸게 했습니다. 프레임을 걷으면 Vue의 onUnmounted가 그 구독을 풀어 줄 거라고 봤습니다. 여닫을수록 메모리는 내려가지 않았고, 부모 창에 건 관측기가 끊기지 않아 iframe 앱이 통째로 남는 쪽이었습니다. 그래서 정리 훅만 믿지 않고, 문서가 숨는 pagehide에서도 같은 정리를 걸어 다시 여닫았습니다. 그때는 메모리가 평평했습니다.

## 프레임을 빼면 정리 훅은 안 돌고, 리스너는 남았습니다

같은 출처의 부모 문서가 iframe을 붙인 뒤, iframe 안의 Vue 3 앱이 마운트되면서 부모 창에 probe 이벤트 리스너와 MutationObserver, PerformanceObserver를 걸게 했습니다. 리스너 콜백은 8MB짜리 배열을 닫아 두었습니다. 부모 문서는 app.unmount()를 부르지 않고 iframe 요소만 제거합니다.

```html
<button type="button" id="close">닫기</button>
<iframe id="child" src="app.html"></iframe>
<script>
document.getElementById('close').onclick = () => {
  document.getElementById('child').remove()
}
</script>
```

```js
import { createApp, onMounted, onUnmounted } from 'vue'

function connect(parent) {
  const payload = new Uint8Array(8 * 1024 * 1024)
  const onProbe = () => { parent.hits = (parent.hits || 0) + 1 }
  parent.addEventListener('probe', onProbe)
  const observer = new MutationObserver(() => {})
  observer.observe(parent.document.body, { childList: true, subtree: true })
  return () => {
    parent.removeEventListener('probe', onProbe)
    observer.disconnect()
  }
}

const app = createApp({
  setup() {
    let disconnect = () => {}
    let cleaned = false
    const cleanup = () => {
      if (cleaned) return
      cleaned = true
      disconnect()
    }
    onMounted(() => { disconnect = connect(window.parent) })
    onUnmounted(() => { cleanup() })
    window.addEventListener('pagehide', () => { cleanup() })
  }
})
app.mount('#app')
```

정리 훅에만 cleanup을 두고 Headless Chrome 157에서 iframe을 제거하자, 기록은 onMounted, pagehide, visibilitychange까지였고 onBeforeUnmount와 onUnmounted, unload는 없었습니다. pagehide의 persisted는 false였습니다. 부모 창에 probe를 다시 보내도 콜백은 이미 실행되지 않았지만, 개발자 도구로 세어 보면 probe 리스너는 그대로 남아 있었습니다. 부모 문서를 바꿔 보아도 MutationObserver 쪽 횟수는 늘지 않았습니다. 같은 여닫기를 12번 반복하면 부모 창에 리스너가 12개 남고, 힙은 0.6MB에서 102.6MB까지 올랐습니다.

같은 정리 함수를 pagehide에서도 부르면, 제거 직후 리스너는 0개였고 12번을 반복한 뒤에도 힙은 약 1MB에서 평평했습니다. 부모가 제거 전에 app.unmount()를 먼저 부르면 Vue 훅이 돌았고, 그때 정리해도 결과는 같았습니다. 이 수치는 실험실에서 잰 값이고, 제품 체감을 대체하지는 않습니다.

## 정리 훅은 Vue가 내릴 때만 돌고, 프레임 제거는 문서 파괴에 가깝습니다

Vue의 onUnmounted는 컴포넌트가 언마운트된 뒤에 수동으로 만든 타이머나 DOM 리스너를 지우라고 되어 있습니다. 그 훅은 Vue가 컴포넌트 트리를 내릴 때 호출됩니다. 부모 문서가 iframe 요소를 제거하는 경로는 Vue의 언마운트가 아닙니다.

WHATWG HTML은 iframe의 제거 절차를 자식 탐색 가능 객체를 파괴하는 일로 적고, 이때 내용 문서는 unload되지 않고 파괴되며 unload 이벤트도 발생하지 않는다고 밝힙니다. 같은 Chrome에서는 같은 출처 iframe을 제거하는 순간에 pagehide와 visibilitychange는 동기적으로 발생했고, unload는 발생하지 않았습니다. 그래서 제거 경로의 정리는 실제로 호출된 문서 수명 이벤트에 두었습니다.

## unload와 visibilitychange, 훅 하나만 두는 길은 버렸습니다

unload에 정리를 두면, 이 Chrome에서는 iframe 제거 때 이벤트 자체가 없었습니다. 공식 문서도 unload를 쓰지 말라고 하고, 뒤로 가기 캐시와 맞지 않으며 브라우저가 기본으로 이 이벤트를 줄이는 쪽이라고 설명합니다.

visibilitychange는 제거 순간에도 hidden으로 와서 정리가 됩니다. 다만 탭만 옮겨도 같은 이벤트가 왔습니다. iframe은 그대로 있는데 부모 창에 건 구독이 풀렸고, 다시 돌아와도 구독은 복구되지 않았습니다. 잠시 숨는 일과 문서가 사라지는 일을 같은 훅으로 다루기엔 범위가 넓었습니다.

정리 훅만 두는 길은, 위에서 본 것처럼 부모 쪽이 요소만 제거할 때 호출되지 않습니다. 반대로 pagehide만 두면, iframe은 남겨 둔 채 Vue만 언마운트할 때 pagehide가 오지 않았습니다. 이때 리스너는 1개 남은 채로 probe에도 계속 응답했습니다. 부모 문서가 제거 전에 unmount를 반드시 부르게 하는 방법은, 부모를 우리가 통제할 때는 확실합니다. 다만 부모 쪽이 요소만 빼는 경우가 있어, 정리의 주된 위치는 iframe 문서의 수명에 두기로 했습니다.

그래서 cleanup은 한 번만 실행되게 가드를 두고, onUnmounted와 pagehide에 같이 걸었습니다. Vue가 정상적으로 내릴 때와, 프레임이 통째로 빠질 때를 한 함수로 맞춥니다.

pagehide도 항상 온다고 보기는 어렵습니다. 모바일에서 다른 앱으로 나갔다가 브라우저를 종료하면 이 이벤트가 오지 않을 수 있고, 명세는 iframe 제거를 언로드가 아니라 파괴로 적습니다. 지금 확인한 것은 이 Chrome에서 같은 출처 iframe을 제거할 때 pagehide가 호출된다는 점입니다. 그 한계를 전제로, 문서가 숨는 신호와 프레임워크 훅을 같이 두는 쪽을 택했습니다.

## 콜백이 안 울리는지만 보면 누수를 놓칩니다

확인은 두 갈래로 나눕니다. 먼저 부모 창에서 같은 이벤트를 다시 보내 콜백이 실행되는지를 봅니다. 다만 이미 파괴된 문서에서 온 리스너는 실행되지 않으면서도 부모 창에 남아 있을 수 있어, 개발자 도구의 getEventListeners로 남은 개수를 같이 셉니다. 그리고 프레임을 여러 번 여닫은 뒤, 가비지 컬렉션을 유도하고 힙이 평평한지 봅니다. 열리는 동안 붙잡은 큰 값이 있다면, 정리되지 않았을 때 힙이 그 크기만큼 쌓이는지가 더 분명합니다.

iframe을 남겨 둔 채 app.unmount()만 호출하는 경로도 같이 봅니다. 이때는 onUnmounted가 돌고 pagehide는 오지 않아야 하며, 그 경로에서도 리스너 개수가 0이 되어야 합니다.

## 남은 구독은 실행되지 않아도, 부모 창의 무게가 됩니다

저는 정리 훅이 부모 창 구독까지 풀어 줄 거라고 보고 관측기를 걸었습니다. 다시 재어 보니 문제는 콜백이 계속 울리는 쪽이 아니었습니다. 프레임을 빼는 순간 훅은 돌지 않았고, 실행되지 않는 리스너가 부모 창에 남아 그 리스너가 닫아 둔 값이 힙을 밀어 올렸습니다. pagehide에 같은 정리를 두자 리스너 개수와 힙이 같이 가라앉았고, 여닫기를 반복해도 메모리는 평평했습니다. 앞선 글에서 숨은 iframe을 미리 불러올지 나눈 것은 켤 조건의 이야기였고, 이번에는 그 프레임을 걷을 때 부모 창에 남은 구독을 푸는 쪽에 가깝습니다.

같은 날 재현하지 못한 버그의 계측 빈칸을 메우는 일과, 조건이 분명한 제안을 코드가 직접 만들게 하는 일은 이 글에서 다루지 않습니다.

부모 창에 건 구독은, 프레임워크 훅과 문서가 숨는 이벤트를 같이 두고 풀려고 합니다.
