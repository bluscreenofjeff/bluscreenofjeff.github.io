---
layout: default
title: Posts Sorted by Tag
permalink: /tags
description: All posts on bluescreenofjeff.com, sorted by subject tag
---

<div class="row">
  <div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1">
<div class="text-center">
	<h1>Posts Sorted by Tag</h1><br>
		
<div class="container-fluid">		
<div class="row">

	{% for tag in site.tags %}
		{% assign t = tag | first %}
		{% assign posts = tag | last %}
		<div class="col-lg-2 col-md-3 col-xs-4" style="padding:3px;"><a href = "#{{t | downcase }}"><button class="btn btn-primary" type="button">{{t | downcase }}</button></a></div>
	{% endfor %}
</div>
</div>
<br>
<div style="text-align:left;">
	{% for tag in site.tags %}
	{% assign t = tag | first %}
	{% assign posts = tag | last %}

<article id="{{t | downcase}}"><br><h2>{{ t | downcase }}</h2>
		{% for post in posts %}
			{% if post.tags contains t %}
			<ul>
				<li>
					<a href="{{ post.url }}">{{ post.title }}</a>
					<span class="date">{{ post.date | date: "%B %-d, %Y"  }}</span>
				</li>
			</ul>

			{% endif %}
		{% endfor %}
		</article>
{% endfor %}

</div>
</div>
</div>
