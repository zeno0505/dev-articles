---
title: "부모 창에 건 리스너를 pagehide에서 풀기"
date: 2026-11-10
categories: [일일 회고]
description: "iframe을 걷을 때 Vue 정리 훅은 안 돌고 부모 창 리스너는 남아, pagehide에서도 같은 정리를 걸었습니다."
---

iframe을 띄웠다가 걷기를 반복할수록 부모 창의 메모리가 내려가지 않았습니다. 성능을 보려고 iframe 안의 앱이 부모 창에 긴 작업 관측기를 걸었고, 프레임을 걷으면 Vue의 onUnmounted가 그 구독을 풀어 줄 거라고 가정했습니다. 부모 창에 건 관측기가 끊기지 않아 iframe 앱이 통째로 남는 쪽이었습니다. 그래서 정리 훅만 믿지 않고, 문서가 숨는 pagehide에서도 같은 정리를 걸고 다시 테스트했습니다. 그때는 몇 번을 반복해도 메모리에 변화가 없었습니다.

이 글은 iframe 안의 앱이 부모 창에 리스너나 관측기를 걸 때, 프레임을 걷은 뒤 메모리가 내려가지 않는 분께 도움이 됩니다.

이 글의 iframe은 마이그레이션 과정에서 채택한 iframe 오버레이입니다. 경로 단위 부분배포로는 기능이 큰 다이얼로그를 옮길 수 없어서, 다른 앱의 다이얼로그를 iframe으로 빌려 썼습니다. 반대 방향도 있습니다. Nuxt4로 옮긴 다이얼로그를 구 화면에서도 그대로 쓰게 하면, 구현을 하나만 유지할 수 있습니다. 채택한 배경은 [Nuxt4를 품는 모노레포에서 경로별 부분배포로 마이그레이션 이어 가기](https://zeno0505.github.io/dev-articles/2026/11/20/partial-deploy-by-path-in-nuxt4-monorepo/)에 정리했습니다.

## 과제와 제약

목표는 부모 창에 건 구독을, 프레임이 사라질 때도 남기지 않는 것입니다. 제약은 세 가지였습니다. 부모 쪽이 요소만 제거할 수 있고, 구독은 iframe 문서가 아니라 부모 창에 남으며, 탭만 옮긴 것처럼 잠시 숨는 일과 문서가 사라지는 일을 같은 훅으로 다루면 안 됩니다.

## 한눈에 보는 결론

- 부모 쪽이 iframe 요소만 제거하면 onUnmounted는 호출되지 않았습니다.
- 이 Chrome에서는 pagehide가 왔고 persisted는 false였으며, unload는 없었습니다.
- 콜백이 실행되지 않아도 리스너는 부모 창에 남아 힙을 밀어 올렸습니다.
- cleanup은 한 번만 실행되게 가드를 두고, onUnmounted와 pagehide에 같이 걸었습니다.

## 제약에 맞춰 고른 방법

![구독은 부모 창에, 정리 훅은 iframe 문서에 걸린다](/dev-articles/assets/2026-11-10-close-with-record-02.png)

정리 훅만 두면, 부모 쪽이 요소만 제거할 때 호출되지 않습니다. pagehide만 두면, iframe은 남겨 둔 채 Vue만 언마운트할 때 이벤트가 오지 않아 리스너가 남고 probe에도 계속 응답했습니다. unload는 이 Chrome에서 iframe 제거 때 이벤트 자체가 없었고, 공식 문서도 쓰지 말라고 합니다. visibilitychange는 iframe을 제거할 때 pagehide 바로 뒤에 hidden으로 왔습니다. 하지만 탭만 옮겨도 같은 hidden이 왔고, 거기서 구독을 끊자 탭으로 돌아와도 관측이 다시 이어지지 않았습니다. 정리하지 않은 관측기는 같은 탭 이동을 거친 뒤에도 계속 기록을 받았으므로, 끊긴 원인은 브라우저가 아니라 hidden에서 건 정리였습니다.

![정리를 거는 자리별로 부모 창 구독이 풀리는지 비교한 표](/dev-articles/assets/2026-11-10-close-with-record-03.png)

그래서 cleanup은 한 번만 실행되게 가드를 두고, onUnmounted와 pagehide에 같이 걸었습니다. 부모 문서가 제거 전에 unmount를 반드시 부르게 하는 방법은, 부모를 우리가 통제할 때는 확실합니다. 다만 부모 쪽이 요소만 빼는 경우가 있어, 정리의 주된 위치는 iframe 문서의 수명에 두기로 했습니다.

## 기본 구조

![부모 창 구독은 두 종료 경로에서 같이 푼다](/dev-articles/assets/2026-11-10-close-with-record-01.png)

Vue의 onUnmounted는 컴포넌트가 언마운트된 뒤에 수동으로 만든 타이머나 DOM 리스너를 지우라고 되어 있습니다. 그 훅은 Vue가 컴포넌트 트리를 내릴 때 호출됩니다. WHATWG HTML은 iframe 제거를 자식 탐색 가능 객체를 파괴하는 일로 적고, 내용 문서는 unload되지 않고 파괴되며 unload 이벤트도 발생하지 않는다고 밝힙니다. 같은 Chrome에서는 같은 출처 iframe을 제거하는 순간에 pagehide와 visibilitychange는 동기적으로 발생했습니다.

실제 고친 모양은, 부모 창 구독을 연결하는 함수와, 한 번만 도는 정리 함수를 훅과 pagehide에 같이 거는 쪽입니다.

```js
import { onMounted, onUnmounted } from 'vue'

export function useParentSubscription(connect) {
  let disconnect = () => {}
  let cleaned = false
  const cleanup = () => {
    if (cleaned) return
    cleaned = true
    disconnect()
    window.removeEventListener('pagehide', cleanup)
  }

  onMounted(() => {
    disconnect = connect(window.parent)
  })
  onUnmounted(cleanup)
  window.addEventListener('pagehide', cleanup)

  return cleanup
}
```

`connect`는 부모 창에 addEventListener나 MutationObserver를 걸고, 푸는 함수를 돌려 주면 됩니다.

## iframe 제거 시 부모 창 리스너가 남는 현상

### 처음 판단: 정리 훅이 구독을 풀어 줄 것

성능을 보려고 부모 창에 관측기를 걸었고, 프레임을 걷으면 정리 훅이 구독을 풀어 줄 거라고 가정했습니다. 반복할수록 메모리는 내려가지 않았고, 부모 창에 건 관측기가 끊기지 않아 iframe 앱이 통째로 남았습니다.

### pagehide에도 둔 정리와 변화 없는 메모리

그래서 문서가 숨는 pagehide에서도 같은 정리를 걸고 다시 테스트했습니다. 그때는 몇 번을 반복해도 메모리에 변화가 없었습니다.

### 울리지 않는 콜백과 남아 있는 리스너

같은 상황을 일반 예시로 다시 재어 보니, 문제는 콜백이 계속 울리는 쪽이 아니었습니다. Headless Chrome 157에서 정리 훅에만 cleanup을 두고 iframe을 제거하자, 기록은 onMounted, pagehide, visibilitychange까지였고 onBeforeUnmount와 onUnmounted, unload는 없었습니다. 부모 창에 probe를 다시 보내도 콜백은 이미 실행되지 않았지만, 개발자 도구로 세어 보면 probe 리스너는 그대로 남아 있었습니다. iframe을 붙였다 떼는 과정을 12번 반복하면 부모 창에 리스너가 12개 남고, 힙은 0.6MB에서 102.6MB까지 올랐습니다. 같은 정리 함수를 pagehide에서도 부르면, 제거 직후 리스너는 0개였고 12번을 반복한 뒤에도 힙은 약 1MB에서 변화가 없었습니다.

## 한계

pagehide도 항상 온다고 보기는 어렵습니다. 모바일에서 다른 앱으로 나갔다가 브라우저를 종료하면 이 이벤트가 오지 않을 수 있고, 명세는 iframe 제거를 언로드가 아니라 파괴로 적습니다. 지금 확인한 것은 이 Chrome에서 같은 출처 iframe을 제거할 때 pagehide가 호출된다는 점입니다. 그 한계를 전제로, 문서가 숨는 신호와 프레임워크 훅을 같이 두는 쪽을 택했습니다.

## 확인 방법

부모 창에서 같은 이벤트를 다시 보내 콜백이 실행되는지를 봅니다. 이미 파괴된 문서에서 온 리스너는 실행되지 않으면서도 부모 창에 남아 있을 수 있어, 개발자 도구의 getEventListeners로 남은 개수를 같이 셉니다. 프레임을 여러 번 붙였다 뗀 뒤 가비지 컬렉션을 유도하고 힙에 변화가 없는지 봅니다. iframe을 남겨 둔 채 app.unmount()만 호출하는 경로도 같이 봅니다. 이때는 onUnmounted가 돌고 pagehide는 오지 않아야 하며, 그 경로에서도 리스너 개수가 0이 되어야 합니다.

저는 정리 훅이 부모 창 구독까지 풀어 줄 거라고 보고 관측기를 걸었습니다. 다시 재어 보니 훅은 돌지 않았고, 실행되지 않는 리스너가 부모 창에 남아 힙을 밀어 올렸습니다. pagehide에 같은 정리를 두자 리스너 개수와 힙이 같이 가라앉았습니다.

## 정리

- 프레임 제거는 Vue 언마운트가 아닙니다.
- 콜백이 안 울리는지만 보면 누수를 놓칩니다.
- 정리 훅과 pagehide를 한 함수로 맞춥니다.

부모 창에 건 구독은, 프레임워크 훅과 문서가 숨는 이벤트를 같이 두고 풀려고 합니다.

## 참고

- [Vue onUnmounted](https://vuejs.org/api/composition-api-lifecycle.html#onunmounted)
- [WHATWG HTML, iframe 제거 절차](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#the-iframe-element)
- [MDN pagehide](https://developer.mozilla.org/en-US/docs/Web/API/Window/pagehide_event)
- [MDN unload](https://developer.mozilla.org/en-US/docs/Web/API/Window/unload_event)
