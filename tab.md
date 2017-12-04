---
layout:     page
title:      "Guitar Tablature"
date:       Sun, 03 Dec 2017 23:32:35 -0500
permalink:  /tab/
---
{% for item in site.tab %}
  <h2>{{ item.title }}</h2>
  <p>{{ item.description }}</p>
  <p><a href="{{ item.url }}">{{ item.title }} Tablature</a></p>
{% endfor %}
