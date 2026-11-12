---
layout: page
title: Articles
permalink: /articles/
---

{% for post in site.posts %}
- {{ post.date | date: "%d %B %Y" }} · [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
