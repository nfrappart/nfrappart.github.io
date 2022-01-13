---
layout: page
title: Terraform
description: all posts about Hashicorp's IaC
---
![Terraform](/pictures/terraform.png)
{% assign posts = site.posts | where_exp: "item", "item.tags contains 'terraform'" %}
{% for post in posts %}
  [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date_to_string }}
{% endfor %}

