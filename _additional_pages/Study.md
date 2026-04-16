---
title: Study
layout: simple
order: 4
---

{% assign grouped = site.study | group_by: "dir" %}
{% for group in grouped %}
## {{ group.name | remove: "/study/" | remove: "/" }}
{% for post in group.items %}
### [{{ post.title | default: post.name }}]({{ post.url }})
{% endfor %}
{% endfor %}
