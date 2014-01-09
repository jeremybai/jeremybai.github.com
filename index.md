---
layout: page
title: 
tagline: 愿有人陪你颠沛流离,如果没有，愿你成为自己的太阳。
---
{% include JB/setup %}
   
## posts列表

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



