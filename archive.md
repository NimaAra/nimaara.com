---
layout: page
title: Blog Archive
---

<link href="{{"/css/override.css" | relative_url}}" rel="stylesheet" type="text/css">

{% for tag in site.tags %}

  <h3>{{ tag[0] }}</h3>
  <ul>
    {% for post in tag[1] %}
      <li><a href="{{ post.url | relative_url }}">{{ post.date | date: "%B %Y" }} - {{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}