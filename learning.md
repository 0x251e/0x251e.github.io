---
layout: page
title: ./category
permalink: /category/
---

## Table of Contents

{% for category in site.categories %}
  <a href="#{{ category[0] | slugify }}">{{ category[0] }}</a>{% unless forloop.last %} • {% endunless %}
{% endfor %}

---

{% for category in site.categories %}
  <h2 id="{{ category[0] | slugify }}">{{ category[0] }}</h2>
  <ul>
    {% for post in category[1] %}
      <li>
        <a href="{{ post.url | relative_url }}">
          {{ post.title }}
        </a>
        <small>{{ post.date | date_to_string }}</small>
      </li>
    {% endfor %}
  </ul>
{% endfor %}
