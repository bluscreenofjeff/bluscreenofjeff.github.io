---
layout: default
title: All Posts
permalink: /posts
featuredimage:
description: All posts on bluescreenofjeff.com
---

<div class="text-center">
	<h1>Post Archive</h1>
	<br/>
</div>

<div class="posts">
<style type="text/css">
<!--
.tab { margin-left: 40px; }
-->
</style>


{% for post in site.posts %}
	<article class="post-preview">
    <a href="{{ post.url }}" class="post-title">
    	<h3>{{ post.title }}</h3>
    	<h4 class="post-subtitle">{{ post.subtitle }}</h4>
    </a>
    <p class="tab"><span class="post-meta">{{ post.date | date: "%B %e, %Y" }}</span> . <span class="post-entry">{{ post.content | truncatewords: 100 | strip_html | xml_escape}}</span><a href="{{ post.url }}"><b> [more]</b></a></p>
    </article>
{% endfor %}


</div>
