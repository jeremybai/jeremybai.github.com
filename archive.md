---
layout: page
title: 归档
permalink: /archive/
banner_image: 
---
---
<div>
  {% for post in site.posts %}
    {% capture currentyear %}{{post.date | date: "%Y"}}{% endcapture %}
    {% if currentyear != year %}
      {% unless forloop.first %}
      </ul>
      {% endunless %}
      <h3>{{ currentyear }}</h3>
      <ul>
      {% capture year %}{{currentyear}}{% endcapture %} 
    {% endif %}
    <li><h4><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a><h4></li>
{% endfor %}
</div>
