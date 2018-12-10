---
layout: page
title: Categories
permalink: /categories
---

<ul class="categoryblock">
{% for category in site.categories %}
  <li><h3><a name="{{ category | first }}">{{ category | first }}</a></h3>
    <ul>
     {% for post in category.last %}
      <li>
        <span class="post-meta">{{post.date | date: "%b %-d, %Y"}}</span>
        <a class="post-link" href="{{ post.url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
    </ul>
  </li>
{% endfor %}
</ul>
