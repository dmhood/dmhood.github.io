---
layout: default
title: Full-Stack Forum
---
{% include JB/setup %}

<div class="blog-index">  
  {% for page in site.posts limit:5 %}
    {% assign page = page %}
    {% assign content = page.content %}
    {% include post_detail.html %}
  {% endfor %}
</div>

[Check out some more posts!]({{site.url}}/archive.html)

{% include JB/commentsCount %}
