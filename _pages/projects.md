---
layout: projects
title: Projects
permalink: /projects/
order: 2
---

## Side projects

{% assign projects = site.projects | where: "type","side" | sort:"date_from" | reverse %}
{% for project in projects %}
{% include project-content.html %}
{% endfor %}

## Full-time projects

{% assign projects = site.projects | where: "type","main" | sort:"date_from" | reverse %}
{% for project in projects %}
{% include project-content.html %}
{% endfor %}

## Jams and hackathons

{% assign projects = site.projects | where: "type","jam" | sort:"date_from" | reverse %}
<div class="row project-grid">
	{% for project in projects %}
		<div class="col col-6 col-t-12">
			{% include project-content.html %}
		</div>
	{% endfor %}
</div>
