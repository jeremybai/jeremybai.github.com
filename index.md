---
layout: page
title: 
tagline: 
---
{% include JB/setup %}

![主页图片]({{site.img_url}}/index.jpg)     
![博物馆旁]({{site.img_url}}/2014-01-09-hello-world-1.jpg) 
##文章列表
<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



