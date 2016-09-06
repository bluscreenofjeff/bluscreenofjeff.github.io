---
layout: default
title: Apache mod_rewrite
permalink: /category/mod_rewrite
---

<div class="text-center">
	<h1>Apache mod_rewrite</h1>
	<br/><br>
	Apache mod_rewrite provides a number of methods to strengthen your phishing and increase the resilience of your testing infrastructure. Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nam mollis lectus ligula. Curabitur tincidunt odio risus, eu luctus ante ullamcorper vel. Sed nec dui nibh. Nunc tellus nisl, bibendum feugiat magna vitae, mattis blandit felis. Quisque congue vel magna sit amet consequat. Nulla eget sapien et tortor commodo convallis vel eget orci. Vestibulum sagittis, tortor id aliquam vehicula, nibh libero laoreet dolor, eu feugiat turpis neque eu diam.


<br><br>
	<ul>
	{% for tag in site.tags %}
		{% assign t = tag | first %}
		{% assign posts = tag | last %}
		{% for post in posts %}
			{% if post.tags contains "mod_rewrite" %}
				<li><a href="{{ post.url }}" >{{ post.title }}</a></li>
			{% endif %}
		{% endfor %}
	{% endfor %}
	</ul>
</div>