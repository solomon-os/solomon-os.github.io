---
layout: default
title: Home
---

# Welcome

This is my personal blog where I share my thoughts and ideas.

## Latest Posts

<ul class="post-list">
{% for post in site.posts limit:5 %}
  <li>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
    <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 150 }}</p>
  </li>
{% endfor %}
</ul>
