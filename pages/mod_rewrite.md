---
layout: default
title: Apache mod_rewrite
permalink: /topics/mod_rewrite
image:
description: Apache mod_rewrite provides a number of methods to strengthen your phishing and increase the resilience of your testing infrastructure. mod_rewrite has the ability to perform conditional redirection based on request attributes, such as URI, user agent, query string, operating system, and IP. Apache mod_rewrite uses htaccess files to configure rulesets for how Apache should handle each incoming request. Using these rules, you could, for instance, redirect requests to your server with the default wget user agent to a legitimate page on your target's website. Many of the techniques discussed on this blog can be combined to increase their effect.
---

<div class="row">
<div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1">
	<h1>Apache mod_rewrite</h1>
	Apache mod_rewrite provides a number of methods to strengthen your phishing and increase the resilience of your testing infrastructure. mod_rewrite has the ability to perform conditional redirection based on request attributes, such as URI, user agent, query string, operating system, and IP. Apache mod_rewrite uses htaccess files to configure rulesets for how Apache should handle each incoming request. Using these rules, you could, for instance, redirect requests to your server with the default wget user agent to a legitimate page on your target's website. Many of the techniques discussed on this blog can be combined to increase their effect.


<h2>Posts About mod_rewrite</h2>
		{% for post in site.posts %}
			{% if post.tags contains "mod_rewrite" %}
				<p><a href="{{ post.url }}" >{{ post.title }}</a> - {{ post.summary }}</p>
			{% endif %}
		{% endfor %}


<h2>Additional Resources</h2>

<ul>
<li><a href="http://mod-rewrite-cheatsheet.com">mod-rewrite-cheatsheet.com</a></li>
<li><a href="http://httpd.apache.org/docs/current/rewrite/">Official Apache 2.4 mod_rewrite Documentation</a></li>
<li><a href="https://httpd.apache.org/docs/2.4/en/rewrite/intro.html">Apache mod_rewrite Introduction</a></li>
<li><a href="http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708">An In-Depth Guide to mod_rewrite for Apache</a></li>
</ul>

</div>
</div>