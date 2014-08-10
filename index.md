---
layout: page
title: 
tagline: Supporting tagline
---
{% include JB/setup %}

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>&nbsp;
						
      <p>{{ post.excerpt | markdownify }}</p>
    </li>
  {% endfor %}
</ul>