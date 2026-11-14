---
title: "정확성 때문에 버린 최적화는 다시 잰다"
date: 2026-11-13
categories: [일일 회고]
---

저장할 본문은 HTML 문자열로 직렬화합니다. 편집 문서 안에서 그 문자열을 만들면, 쌓여 있는 Range를 브라우저가 같이 다루느라 입력 한 자가 무거워질 수 있어, 그 비용을 피하려고 document.implementation.createHTMLDocument로 분리된 문서를 만들고 그쪽에서 읽으려 했습니다. 예전에 이 길을 버린 이유는, 저장되는 문자열이 달라질 수 있다는 점이었습니다.

## 분리된 문서는 기준 주소와 직렬화 방법이 다를 수 있다

Tiptap의 getHTML은 편집 중인 문서를 HTML로 돌려 줍니다. 그 아래의 ProseMirror DOMSerializer.serializeFragment는 document 옵션을 받을 수 있습니다. 공식 문서는 브라우저가 아닐 때 노드를 만들 문서를 넘기라고 적습니다.

createHTMLDocument가 만드는 문서는 지금 페이지와 분리되어 있고, 기준 주소는 about:blank입니다. 같은 태그를 만들더라도, 상대 주소의 속성값 대신 요소의 href나 src 프로퍼티를 읽어 저장하면 기준 주소를 따라 절대 주소로 풀립니다. XMLSerializer.serializeToString은 DOM을 XML 형태로 직렬화하므로, innerHTML이나 outerHTML으로 읽은 HTML과 같은 문자열이 아닙니다. innerHTML처럼 속성 그대로 읽는 경로에서는 두 문서의 결과가 같을 수 있고, 갈리는지는 읽기 방법을 나눠 봐야 합니다.

## innerHTML은 같고, 주소 프로퍼티와 XMLSerializer는 갈렸다

같은 현상이 나는 최소 예시는 아래와 같습니다. Headless Chrome 156에서 실행해 확인했습니다.

```html
<base href="https://example.com/posts/1">
<div id="article">
  <p><a href="/about">상대 링크</a> <img src="/cover.png" alt="표지"></p>
  <p>줄 바꿈<br></p>
</div>
```

```js
function htmlByInner(source, doc) {
  const wrap = doc.createElement('div')
  wrap.appendChild(doc.importNode(source, true))
  return wrap.innerHTML
}

function htmlByResolvedUrl(source, doc) {
  const wrap = doc.createElement('div')
  wrap.appendChild(doc.importNode(source, true))
  wrap.querySelectorAll('a[href]').forEach((a) => {
    a.setAttribute('href', a.href)
  })
  wrap.querySelectorAll('img[src]').forEach((img) => {
    img.setAttribute('src', img.src)
  })
  return wrap.innerHTML
}

const article = document.querySelector('#article')
const liveInner = htmlByInner(article, document)

const detached = document.implementation.createHTMLDocument('')
const detachedInner = htmlByInner(article, detached)
```

liveInner와 detachedInner는 같았습니다. 상대 주소 속성은 두 문서 모두 `/about`, `/cover.png`로 남았습니다. 갈린 것은 다음 두 경로뿐입니다.

href와 src 프로퍼티를 읽어 속성에 다시 쓰면, 편집 문서에서는 `https://example.com/about`, `https://example.com/cover.png`가 되고, 분리된 문서에서는 `/about`, `/cover.png`로 남았습니다. XMLSerializer로 `br`을 읽으면 두 문서 모두 `<br xmlns="http://www.w3.org/1999/xhtml" />`가 되고, outerHTML은 `<br>`입니다. 화면은 같아 보여도, 저장 문자열은 이 읽기 방법에서만 달라졌습니다.

## 저장 결과가 같은지와 한 자 입력으로 확인한다

그래서 이 최적화를 다시 넣을 때는, innerHTML처럼 같았던 경로인지부터 가릅니다. 굵게, 기울임, 링크, 이미지, 목록처럼 실제로 쓰는 서식을 한 번씩 넣고, 저장한 뒤 다시 열었을 때 화면과 문자열이 같은지 확인하면 됩니다. href·src를 절대 주소로 풀어 쓰거나 XMLSerializer로 저장하는 경로가 있으면, 그 차이까지 대조해야 합니다.

입력 체감은 스크립트 시간만으로 단정하지 않습니다. 손으로 쌓기 어려운 무거운 상태는 콘솔로만 만들어 두고, 좋아졌는지는 사람이 글자를 한 자 쳐서 봅니다. 같은 절차를 수정 전과 수정 후에 반복합니다.

같은 Chrome에서 단락 350개와 Range 8만 개를 편집 문서에 남긴 뒤, 그 문서로 노드를 옮기며 innerHTML을 읽는 경로와 분리된 문서에서 읽는 경로를 10번 비교했습니다. 중앙값은 각각 3ms, 0.2ms였습니다. 두 경로의 innerHTML은 길이가 같았습니다. 이 숫자는 실험용 페이지의 결과이고, 실제 입력 체감을 대신하지는 않습니다.

## 속도만 보고 다시 넣지는 않는다

선택지는 세 가지였습니다. 첫째, 분리된 문서에서 HTML을 만들되, 저장 결과가 같은 경로인지를 확인하고 쓰는 것입니다. 둘째, 예전에 버린 이유만 믿고 이 방식을 계속 닫아 두는 것입니다. 셋째, 체감이 좋아진 것만 보고 출력 비교 없이 넣는 것입니다.

계속 닫아 두는 길은 버렸습니다. innerHTML은 두 문서에서 같았고, 갈린 지점은 확인된 읽기 방법뿐이었습니다. 출력 비교 없이 넣는 길도 버렸습니다. 입력은 가벼워져도, 저장된 HTML이 바뀌면 미리보기와 발행에서 구멍이 납니다.

얻는 것은 입력 한 자가 멈추는 시간을 줄일 여지입니다. 치르는 것은 직렬화 경로를 바꿀 때마다, 같았던 경로와 갈린 경로를 나눠 저장 문자열을 대조해야 한다는 점입니다.

## 같은 경로를 확인한 뒤에 다시 쓴다

저는 innerHTML로 읽은 결과는 두 문서에서 같았고, 갈린 것은 href·src 프로퍼티와 XMLSerializer뿐이라는 점을 본 뒤에야, 이 방식을 다시 쓸 수 있다고 봅니다. 실험용 페이지에서는 분리된 문서 쪽이 더 짧았고, 입력 한 자의 멈춤이 줄어드는 것도 확인했습니다. 그 실험 숫자는 제품 체감을 대신하지 않으므로, 저장 문자열이 같은지를 먼저 보고 한 자 입력으로 확인한 뒤에 쓰려고 합니다. 줄이기 전에 무게를 재자고 했던 일과는 층이 다릅니다. 이번에는 이미 버려 둔 방식을, 실제로 갈린 경로와 같았던 경로를 나눠 본 뒤에 다시 여는 일입니다.

구형 브라우저 지원을 실제 사용자 수로 정하는 일과, 새 입력 방식을 붙이기 전에 저장 형식부터 보는 일은 다음 글감으로 남깁니다.

정확성 때문에 버린 최적화는, 버린 이유와 지금 출력부터 다시 재려고 합니다.
