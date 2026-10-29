---
layout: default
permalink: /
title: Posts
---
# Posts

<ul class="post-list" style="list-style:none;padding-left:0;">
  {% for post in site.posts %}
  <li style="margin:.5rem 0;">
    <a href="{{ post.url }}">{{ post.title }}</a>
    <div class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</div>
  </li>
  {% endfor %}
</ul>
