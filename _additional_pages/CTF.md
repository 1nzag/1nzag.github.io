---
title: CTF
layout: simple
order: 3
---

{% assign grouped = site.ctf | group_by: "dir" %}
{% for group in grouped %}
## {{ group.name | remove: "/_ctf/" | remove: "/ctf/" | remove: "/" | replace: "_", " " }}
{% for post in group.items %}
+ [{{ post.title | default: post.name }}]({{ post.url }})
{% endfor %}
{% endfor %}