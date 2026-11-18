---
layout: default
permalink: /insight/
title: 인사이트
---
<section class="intro">
  <h1>인사이트</h1>
  <p>{{ site.description | remove: "일일 회고와 인사이트 — " }}</p>
  <p class="intro-about">발표와 작업에서 정리한 생각을 글로 남깁니다.</p>
</section>
<ol class="post-list" reversed>
  {% for post in site.posts %}
  {% if post.categories contains "인사이트" %}
  <li>
    <article>
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y. %-m. %-d." }}</time>
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      {% if post.description and post.description != "" %}{% assign summary = post.description %}{% else %}{% assign summary = post.excerpt %}{% endif %}
      {% assign summary = summary | strip_html | normalize_whitespace | strip | truncate: 140 %}
      {% if summary != "" %}<p class="excerpt">{{ summary }}</p>{% endif %}
    </article>
  </li>
  {% endif %}
  {% endfor %}
</ol>
