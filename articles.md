---
layout: default
title: Articles
permalink: /articles/
---

<h1>Articles</h1>
<p class="lead">Modern C++ and embedded-systems engineering, explained through concrete experiments.</p>

{% for post in site.posts %}
<article class="list-item">
  <p class="meta">{{ post.date | date: "%B %-d, %Y" }}</p>
  <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
  <p>{{ post.description }}</p>
</article>
{% endfor %}
