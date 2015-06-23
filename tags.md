---
layout: page
title: Blog Tags
permalink: /tags/
banner_image: sample-banner-image-3.jpg
---

{% for tag in site.tags %} 
  <h2 id="{{ tag[0] }}-ref">{{ tag[0] }}</h2>
  <ul>
    {% assign pages_list = tag[1] %}  
  </ul>
{% endfor %}