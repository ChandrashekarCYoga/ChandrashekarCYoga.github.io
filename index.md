---
layout: default
title: Home
---

<section class="hero">
  <p class="eyebrow">Embedded architecture · Real-time systems · Modern C++</p>
  <h1>Engineering abstractions that survive contact with hardware.</h1>
  <p class="lead">
    Practical notes on predictable firmware, zero-overhead abstractions,
    RTOS design, testing, performance, and architecture.
  </p>
  <div class="actions">
    <a class="button" href="/articles/">Read the articles</a>
    <a class="button secondary" href="https://github.com/{{ site.author.github }}">View experiments</a>
  </div>
</section>

<section>
  <h2>Latest writing</h2>
  <div class="cards">
    {% for post in site.posts limit: 6 %}
    <article class="card">
      <p class="meta">{{ post.date | date: "%B %-d, %Y" }}</p>
      <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
      <p>{{ post.description }}</p>
    </article>
    {% endfor %}
  </div>
</section>

<section class="principles">
  <h2>What you will find here</h2>
  <div class="cards">
    <article class="card"><h3>Design decisions</h3><p>Alternatives, trade-offs, failure modes, and rationale.</p></article>
    <article class="card"><h3>Generated evidence</h3><p>Assembly, code size, timing data, and reproducible builds.</p></article>
    <article class="card"><h3>Embedded context</h3><p>Determinism, memory limits, interrupts, concurrency, and hardware constraints.</p></article>
  </div>
</section>
