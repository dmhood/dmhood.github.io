---
layout: default
title: Full-Stack Forum
---
{% include JB/setup %}

<div class="blog-index">  
  {% for page in site.posts %}
    {% assign page = page %}
    {% assign content = page.content %}
    {% include post_detail.html %}
  {% endfor %}
</div>
