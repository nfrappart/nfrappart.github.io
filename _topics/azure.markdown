---
layout: page
title: Azure
description: all posts about Microsoft Azure
---
![Azure](/pictures/azure.png)
{% assign posts = site.posts | where_exp: "item", "item.tags contains 'azure'" %}
{% for post in posts %}
  [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date_to_string }}
{% endfor %}

