---
layout: page
title: Blog Posts
permalink: /blog/
---

All lab journal entries and technical write-ups.

---

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url }})
**{{ post.date | date: "%B %d, %Y" }}** {% if post.categories %} | {{ post.categories | join: ", " }}{% endif %}

{{ post.excerpt | strip_html | truncate: 200 }}

---
{% endfor %}
