---
layout: page
title: 2 Bit Hacker
tagline: random thoughts and hackery
---
{% for post in site.posts %}
{{ post.title }}
======

{{ post.content }}

Posted: {{ post.date | date_to_string }} [Link]({{ post.url }})

{% endfor %}
