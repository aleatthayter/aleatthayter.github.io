---
layout: default
title: Home
---

{% for post in site.posts %}
  <div class="post">
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
    <span class="post-date">{{ post.date | date_to_string }}</span>
    <p>{{ post.excerpt }}</p>
  </div>
{% endfor %}