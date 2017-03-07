---
layout: archive
permalink: /other/
title: "Latest Posts *other*"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'other' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->
