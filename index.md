---
layout: default
title: Home
---

> Hi. My name is Ben "Young" Mwanzia. Welcome to my blog.

{% for post in site.posts limit:5 %}
### {{ post.date | date_to_long_string }} <a href="{{ post.url }}">{{ post.title }}</a>
{{ post.excerpt }}
{% endfor %}

### [More...](/archives/)
