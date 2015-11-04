---
layout: page
title: 分类
permalink: /categories/
banner_image: sample-banner-image-3.jpg
---
---
<div>
{% for category in site.categories %}
	<a style="line-height:30px; font-size:20px;" href="#{{ category[0] }}" rel="{{ category[1].size }}">{{ category[0] }}({{ category[1].size }})</a>&nbsp;&nbsp;
{% endfor %}
</div>
---
<div>
{% for category in site.categories %}
<h2><a id="{{ category[0] }}" name="{{ category | first }}">{{ category | first }}</a></h2>
	<ul>
	{% for posts in category %}
		{% for post in posts %}
			{% if post.url %}
				<li><h4><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a><h4></li>
			{% endif %}
		{% endfor %}
	{% endfor %}
	</ul>
{% endfor %}
</div>
