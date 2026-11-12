---
layout: page
title: Articles
permalink: /articles/
---

# All articles

<div class="article-grid">
{% for post in site.posts %}
  <article class="article-card">
    <div class="card-category">{{ post.categories | join: " · " }}</div>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p>{{ post.description | default: post.excerpt | strip_html | truncate: 180 }}</p>
    <div class="card-meta">{{ post.date | date: "%d %B %Y" }}</div>
    <a class="read-more" href="{{ post.url | relative_url }}">Read article →</a>
  </article>
{% endfor %}
</div>
