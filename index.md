---
layout: default
---
# Welcome to Gears and Grind!

Here are my latest posts:

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}
