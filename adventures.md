---
layout: default
title:  "adventures"
pemalink: "adventures"
---
# adventures

{% for post in site.categories.adventures %}
 <span>{{ post.date | date_to_string }}</span> &nbsp; <b><a href="{{ post.url | relative_url }}">{{post.title}}</a></b><br>{{ post.content | markdownify | strip_html | truncatewords: 50 }}
{% endfor %}
