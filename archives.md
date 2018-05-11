---
layout: default
permalink: /archives/
author_profile: true
use-site-title: true
---

## The Archives
{% for post in site.posts %}
* {{ post.date | date_to_string }} - [{{ post.title }}]({{ post.url }})
{% endfor %}
