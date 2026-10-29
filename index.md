---
layout: default
permalink: /
title: Posts
---
<div class="posts">
  {% for post in site.posts %}
  <article class="post-card">
    {% if post.image %}
    <a class="post-thumb" href="{{ post.date | date: '%Y/%m' }}/{{ post.slug }}/"><img src="{{ post.image }}" alt=""></a>
    {% endif %}
    <div class="post-card-body">
      <h2 class="post-card-title"><a href="{{ post.date | date: '%Y/%m' }}/{{ post.slug }}/">{{ post.title }}</a></h2>
      <div class="post-date">{{ post.date | date: "%Y-%m-%d" }}</div>
      {% if post.description %}<p class="post-desc">{{ post.description }}</p>{% endif %}
    </div>
  </article>
  {% endfor %}
</div>
