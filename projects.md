---
layout: default
title:  "projects"
pemalink: "projects"
---
# projects

{% for post in site.categories.projects %}
 <span>{{ post.date | date_to_string }}</span> &nbsp; <b><a href="{{ post.url }}">{{ post.title }}</a></b><br>{{ post.content | markdownify | strip_html | truncatewords: 50 }}
{% endfor %}
