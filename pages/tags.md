---
layout: default
title: Posts Sorted by Tag
permalink: /tags
image:
description: All posts on bluescreenofjeff.com, sorted by subject tag
---
<style>
.site-tag a {
    display: inline-block;
    margin-right: 11px;
}
</style>
<div class="row">
  <div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1">
<div class="text-center">
	<h1>Posts Sorted by Tag</h1><br>
		
<div class="container-fluid">		
<div class="row">

{% assign tags = site.tags | sort %}
{% for tag in tags %}
 <span class="site-tag">
    <a href="#{{ tag | first | downcase }}"
        style="font-size: {{ tag | last | size  |  times: 12 | plus: 80  }}%">
            {{ tag[0] | replace:'-', ' ' }}
    </a>
</span>
{% endfor %}

</div>
</div>
<br>
<div style="text-align:left;">
	{% for tag in site.tags %}
	{% assign t = tag | first %}
	{% assign posts = tag | last %}

<article id="{{t | downcase}}" style="margin-top: 0;"><br><h2>{{ t | downcase }}</h2>
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
