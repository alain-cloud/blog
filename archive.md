---
layout: page
title: Articles
permalink: /articles/
---

<p class="archive-intro">Browse {{ site.posts.size }} practical guides and field notes by topic. Each section gathers related articles so you can find the right starting point quickly.</p>

{% assign topic_groups = site.posts | group_by_exp: "post", "post.categories.first | downcase | default: 'general'" | sort: "name" %}
<nav class="archive-topics" aria-label="Article topics">
  {% for group in topic_groups %}
    <a href="#{{ group.name | slugify }}">{% include topic-label.html name=group.name %} <span aria-hidden="true">{{ group.items.size }}</span></a>
  {% endfor %}
</nav>

{% for group in topic_groups %}
  <section class="archive-group" id="{{ group.name | slugify }}" aria-labelledby="heading-{{ group.name | slugify }}">
    <h2 id="heading-{{ group.name | slugify }}">{% include topic-label.html name=group.name %} <span>{{ group.items.size }} {% if group.items.size == 1 %}article{% else %}articles{% endif %}</span></h2>
    <div class="article-grid">
      {% for post in group.items %}
        <article class="article-card">
          <div class="card-meta"><time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%d %B %Y" }}</time></div>
          <h3><a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a></h3>
          <p>{{ post.description | default: post.excerpt | strip_html | normalize_whitespace | truncate: 170 }}</p>
          <a class="read-more" href="{{ post.url | relative_url }}">Read article&nbsp; →</a>
        </article>
      {% endfor %}
    </div>
  </section>
{% endfor %}
