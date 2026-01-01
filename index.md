---
layout: default
title: Home
---

# Welcome

I'm Solomon, a software engineer exploring the intersection of high-performance computing and quantitative finance. This blog documents my technical journey.

## Latest Posts

<ul class="post-list">
{% for post in site.posts limit:5 %}
  <li>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
    <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 200 }}</p>
  </li>
{% endfor %}
</ul>

{% if site.posts.size == 0 %}
<p style="color: var(--text-muted);">No posts yet. Check back soon!</p>
{% endif %}
