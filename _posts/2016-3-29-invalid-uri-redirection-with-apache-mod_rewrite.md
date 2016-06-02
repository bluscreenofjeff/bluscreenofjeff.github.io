---
layout: post
title: Invalid URI Redirection with Apache mod_rewrite
summary: How to redirect users visiting non-existent file paths in your phishing infrastructure to a different site.
tags: 
- mod_rewrite
- phishing
---

There have been times when a curious phish recipient or a zealous help desk staff has loaded the phishing link in their browser and decided to take a peek at a higher directory or the root domain. Of course, most times there isn't much else site to see. In those cases, the chances of being reported to IR went up significantly, sometimes leading to a phishing campaign being blocked. This is where invalid URI redirection comes in handy.

We can whitelist resources the Apache server will proxy for the targets and redirect any other requests to the target's real domain or another page of our choosing.

In the demo below, the user navigates to *spoofdomain.com/really/long/url.html* and is served a page; however, when the user navigates to *spoofdomain.com/really/* the browser is redirected to *google.com*.

![Invalid URI Redirection Demo](/assets/apache/invalid-uri-demo.gif)

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/(profiler|payload)/?$</span> <span style="color: mediumpurple">[NC,OR]</span><br>
RewriteCond <span style="color: dodgerblue">%{HTTP_REFERER}</span> <span style="color: mediumseagreen">^http://SPOOFED-DOMAIN\.com</span> <span style="color: mediumpurple">[NC]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REDIRECTION-URL.com/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
</div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's URI </span> <span style="color: mediumseagreen">is either '/profiler' or '/payload' (with an optional trailing slash), </span> <span style="color: mediumpurple"> ignoring case; OR </span><br>
<span style="color: dodgerblue">If the request's referer </span> <span style="color: mediumseagreen">starts with 'http://SPOOFED-DOMAIN.com', </span> <span style="color: mediumpurple">ignoring case</span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span><span style="color: orange">from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
If the above conditions are not met, <span style="color: gold">change the entire request</span> <span style="color: orange"> to http://REDIRECTION-URL.com/ </span><span style="color: mediumvioletred">and drop any query strings from the original request. </span> <span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span>
</div>

There are two handy mod_rewrite regex strings being used in this ruleset. The first is the `/?$` string on line two. In the RewriteCond context, the question mark indicates that the previous character, the trailing slash in the example, is optional. Without this regex, RewriteCond would only match the path exactly as written in the ruleset. The dollar sign signifies the end of the URI, meaning the request */payload1* would not match.

The second useful regex is the question mark in the last line. In the RewriteRule context, the question mark tells Apache to drop the query string from the redirected request. 

Since the Proxy `[P]` flag is set on the first RewriteRule, the address bar will still show the original domain name and just append the requested URI to the end of the address. Conversely, if the request doesn't match either condition, the request will be redirected and the address bar will update and show companydomain.com.


A similar use for this redirection would be to redirect all request URIs to one payload. For instance, if you were phishing users in different departments and each email scenario's link was unique you could redirect all users to a single payload without needing to stand up separate copies. To achieve this, simply remove the `$` from the first RewriteCond line. Now the rule will match */profiler* and */profiler/humanresources/legitimatepage.html*.

Redirecting requests with invalid URIs can help a phishing website pass the sniff test for prying recipients or IT. By redirecting non-existent resources users won't reach any index listings or 404 errors on pages that should logically exist. 

# Strengthen Your Phishing with Apache mod_rewrite Posts

* [Strengthen Your Phishing with Apache mod_rewrite and Mobile User Redirection]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/)
* Invalid URI Redirection
* [Operating System Based Redirection with Apache mod_rewrite]({{site.baseurl}}/2016-04-05-operating-system-based-redirection-with-apache-mod_rewrite/)
* [Combatting Incident Responders with Apache mod_rewrite]({{site.baseurl}}/2016-04-12-combatting-incident-responders-with-apache-mod_rewrite/)
* [Expire Phishing Links with Apache RewriteMap]({{site.baseurl}}/2016-04-19-expire-phishing-links-with-apache-rewritemap/)

{:#resources}

# Resources

* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
	* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)