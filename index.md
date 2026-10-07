# zeno-articles

일일 회고를 짧게 남기는 개인 기록.

글은 `_posts`에 있고, 카테고리는 **일일 회고**다.

## 최근 글

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%Y년 %m월 %d일" }}
{% endfor %}
