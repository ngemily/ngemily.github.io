---
layout: page
title: Index
permalink: /post-index/
---
{% if site.paginate %}
  {% assign posts = paginator.posts %}
{% else %}
  {% assign posts = site.posts %}
{% endif %}


<ul class="post-index-list">
  {%- assign date_format = site.minima.date_format | default: "%Y-%m-%d" -%}
  {%- for post in posts -%}
  <li>
    <span class="post-meta date">{{ post.date | date: date_format }}</span>
    <a class="post-meta" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
  </li>
  {%- endfor -%}
</ul>
