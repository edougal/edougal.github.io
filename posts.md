---
title: Posts
layout: archive
author_profile: true
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}
      | {{ post.date | date: '%B %d, %Y' }}</a>
      <br> 
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>