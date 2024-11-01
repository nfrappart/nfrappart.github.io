---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults
title: HashiCorp on Azure & Other funny stuff
layout: home
---

## What's this site ?
This is a nerdy blog about Cloud related tech.
A youtube channel is in the making, it just takes more time ^_^'

## Pages
{% for topic in site.topics %}
- [{{ topic.title }}]({{ topic.url | relative_url }})
{% endfor %}

## Posts 
{% for post in site.posts %}
- {{ post.date | date_to_string }}: [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
