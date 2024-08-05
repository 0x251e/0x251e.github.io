---
layout: default
title: Learning Resources
---

<h1>{{ page.title }}</h1>

<ul>
  {% for post in site.posts %}
    {% if post.categories contains "learning" %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>
        <p>{{ post.excerpt }}</p>
      </li>
    {% endif %}
  {% endfor %}
</ul>

