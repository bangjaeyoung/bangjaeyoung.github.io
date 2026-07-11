---
icon: fas fa-code
order: 2
title: Dev
---

<ul class="content ps-0">
{% assign posts = site.categories["Dev"] %}
{% for post in posts %}
  <li class="d-flex justify-content-between px-md-3">
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>

### Troubleshooting

<ul class="content ps-0">
{% assign ts_posts = site.categories["Troubleshooting"] %}
{% for post in ts_posts %}
  <li class="d-flex justify-content-between px-md-3">
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>
