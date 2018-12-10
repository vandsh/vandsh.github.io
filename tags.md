---
layout: page
title: Tags
permalink: /tags
---

<ul class="tagblock">
{% for tag in site.tags %}
  <li><h3><a name="{{ tag | first }}">{{ tag | first }}</a></h3>
    <ul>
    {% for post in tag.last %}
      <li>
        <span class="post-meta">{{post.date | date: "%b %-d, %Y"}}</span>
        <a class="post-link" href="{{ post.url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
    </ul>
  </li>
{% endfor %}
</ul>
