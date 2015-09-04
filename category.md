---
layout: default
title: category
---

<div class="well">
{% capture categories %}{% for category in site.categories %}{{ category | first }}{% unless forloop.last %},{% endunless %}{% endfor %}{% endcapture %}
{% assign category = categories | split:',' | sort %}
{% for item in (0..site.categories.size) %}{% unless forloop.last %}
{% capture word %}{{ category[item] | strip_newlines }}{% endcapture %}
<h2 class="category" id="{{ word }}">{{ word }}</h2>
<ul class="categorys">
{% for post in site.categories[word] %}{% if post.title != null %}
<li>
<span class="date">{{ post.date | date_to_string }} >> </span> <a href="{{ site.baseurl}}{{ post.url }}">{{ post.title }}</a>
</li>
{% endif %}{% endfor %}
</ul>
{% endunless %}{% endfor %}
<br/><br/>
</div>