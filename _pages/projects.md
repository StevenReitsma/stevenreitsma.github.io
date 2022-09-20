---
layout: projects
title: Projects
permalink: /projects/
order: 2
---

## Side projects

{% assign projects = site.projects | where: "type","side" | sort:"date_from" | reverse %}
<div class="container">
	{% for project in projects %}
	<div class="row" style="width: 100%">
		<div class="col" style="width: 100%">
			{% include project-content.html %}
		</div>
	</div>
	{% endfor %}
</div>

## Full-time projects

{% assign projects = site.projects | where: "type","main" | sort:"date_from" | reverse %}
<div class="container">
	{% for project in projects %}
		<div class="row" style="width: 100%">
			<div class="col" style="width: 100%">
				{% include project-content.html %}
			</div>
		</div>
	{% endfor %}
</div>

## Jams and hackathons

{% assign projects = site.projects | where: "type","jam" | sort:"date_from" | reverse %}
<div class="row">
	{% for project in projects %}
		<div class="col col-6 col-t-12 col-m-12">
			{% include project-content.html %}
		</div>
	{% endfor %}
</div>
