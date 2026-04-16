---
title: CTF
layout: simple
order: 3
---

{% assign grouped = site.ctf | group_by_exp: "post", "post.relative_path | split: '/' | slice: 1 | first" %}
{% for group in grouped %}
## {{ group.name | replace: "_", " " }}
{% for post in group.items %}
+ [{{ post.title | default: post.name }}]({{ post.url }})
{% endfor %}
{% endfor %}