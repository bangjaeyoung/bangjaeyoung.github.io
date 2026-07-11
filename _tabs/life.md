---
icon: fas fa-mug-hot
order: 1
title: Life
---

<ul class="content ps-0">
{% assign posts = site.categories["Life"] %}
{% for post in posts %}
  <li class="d-flex justify-content-between px-md-3">
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>
