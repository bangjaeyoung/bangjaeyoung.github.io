---
icon: fas fa-mug-hot
order: 1
title: Life
---

{% assign posts = site.categories["Life"] %}

<div id="post-list" class="flex-grow-1 px-xl-1">
  {% for post in posts %}
    <article class="card-wrapper card">
      <a href="{{ post.url | relative_url }}" class="post-preview row g-0 flex-md-row-reverse">
        <div class="col-md-12">
          <div class="card-body d-flex flex-column">
            <h1 class="card-title my-2 mt-md-0">{{ post.title }}</h1>

            <div class="card-text content mt-0 mb-3">
              <p>{% include post-summary.html %}</p>
            </div>

            <div class="post-meta flex-grow-1 d-flex align-items-end">
              <div class="me-auto">
                <i class="far fa-calendar fa-fw me-1"></i>
                {% include datetime.html date=post.date %}
              </div>
            </div>
          </div>
        </div>
      </a>
    </article>
  {% endfor %}
</div>
