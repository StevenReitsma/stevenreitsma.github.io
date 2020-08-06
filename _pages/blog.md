---
layout: default
title: Blog
permalink: /blog/
---

<div class="container">
	<div class="row">
		<div class="col col-12">
			{% if site.posts.size > 0 %}
				<h4 class="lates-title">All Posts</h4>
			{% endif %}
		</div>
	</div>
</div>

<div class="container">
	<div class="row">
	{% if site.posts.size > 0 %}
		{% for post in site.posts %}
			{% include article-content.html %}
		{% endfor %}
	{% endif %}
	</div>
</div>