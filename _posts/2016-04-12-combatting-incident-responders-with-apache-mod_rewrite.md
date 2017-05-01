---
layout: post
title: Combatting Incident Responders with Apache mod_rewrite
summary: Tricks to slow down and impede incident responders investigating your phishing sites.
tags: 
- mod_rewrite
- phishing
commentIssueId: 10
---

Any phishing campaign involving an active incident response element usually requires some evasive steps to prolong its longevity. This often includes being stealthier, performing anti-forensics actions, or avoiding certain tradecraft altogether. Phishing is no different, and is often the most 'vulnerable' part of a campaign from an active IR perspective. Using a distributed infrastructure built with independent components helps reduce the risk of the overall architecture being blocked, but individual phishing campaigns are likely to be caught and blocked throughout the duration. The longer we can stretch out the usability of each of those campaigns, the better our chances of gaining access.

Using [Apache mod_rewrite]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/) rules, we can rewrite potential incident responder or security appliance requests to an innocuous website or the target's real website. While the methods discussed below won't stave off a concerted investigation, it will hopefully make the malicious website pass the 'sniff test' with recipients and lower level help desk or incident responders. 

It's important to note that these techniques could prevent valid phishing victims from reaching your malicious website. You will need to weigh the risks of losing out on potential clicks against the risks posed by the target's incident responders.

# Block Common IR User Agents

Blocking common incident responder or security appliance user agents is an easy way to reject a swath of traffic hitting your malicious website. I watch my web server's Apache access logs like a hawk while a phishing campaign is active. It usually doesn't take long before web crawlers and security appliances start making GET requests. That traffic is normal for web servers, but we don't really want our malicious website being crawled, classified, and archived when we're trying to be stealthy. 

Add the following ruleset to the top of any ruleset already in place, below the `RewriteEngine On` line, to redirect any blank or provided user agents to an alternate location. If you identify the security products your target uses and think it is performing heuristic analysis on the page for filtering purposes, you can additionally add a separate copy of these rules to match the product(s) and proxy the request to the alternate site, rather than redirect. 

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteCond <span style="color: dodgerblue">%{HTTP_USER_AGENT}</span> <span style="color: mediumseagreen">"wget|curl|HTTrack|crawl|google|bot|b\-o\-t|spider|baidu"</span> <span style="color: mediumpurple">[NC,OR]</span><br>
RewriteCond <span style="color: dodgerblue">%{HTTP_USER_AGENT}</span> <span style="color: mediumseagreen">=""</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://COMPANYDOMAIN.com/404.html/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
</div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
<span style="color: dodgerblue">If the request's user agent</span> <span style="color: mediumseagreen">matches any of the provided keywords, </span> <span style="color: mediumpurple">ignoring case. OR</span><br>
<span style="color: dodgerblue">If the request's user agent</span> <span style="color: mediumseagreen">is blank</span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to http://COMPANYDOMAIN.com/404.html/ </span><span style="color: mediumvioletred">and drop any query_strings from original request. </span><span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span><br>
</div>

As mentioned in my first post about mod_rewrite, placing this ruleset in a *.htaccess* file allows changes to be made on the fly without needing to restart the Apache service. We can make adjustments as the campaign progresses if we notice more user agents that should be filtered. For a lengthy list of common user agents, check out [useragentstring.com](http://www.useragentstring.com/pages/useragentstring.php).


# IP Filtering

IP filtering provides the ability to selectively proxy requests or redirect the user based upon the originating IP address. We can monitor the Apache logs and modify filtering rules throughout the phishing campaign's execution.

## Whitelisting
IP whitelisting is an option if you can determine the IP addresses or ranges from which the targets will originate. For straight whitelisting, use the following ruleset:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REMOTE_ADDR}</span> <span style="color: mediumseagreen">^100\.0\.0\.</span> <span style="color: mediumpurple">[OR]</span><br>
RewriteCond <span style="color: dodgerblue">%{REMOTE_ADDR}</span> <span style="color: mediumseagreen">^100\.0\.1\.</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://COMPANYDOMAIN.com/404.html/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
</div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the requestor's IP address </span> <span style="color: mediumseagreen">starts with 100.0.0, </span> <span style="color: mediumpurple"> ; OR </span><br>
<span style="color: dodgerblue">If the requestor's IP address </span> <span style="color: mediumseagreen">starts with 100.0.1, </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span><span style="color: orange">from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
If the above conditions are not met, <span style="color: gold">change the entire request</span> <span style="color: orange"> to http://COMPANYDOMAIN.com/404.html </span><span style="color: mediumvioletred">and drop any query strings from the original request. </span> <span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span>
</div>

In this example, any user visiting from 100.0.0.0/24 and 100.0.1.0/24 will have the original URI request proxied to the teamserver. Visitors from other IPs will be redirected to *companydomain.com/404.html*. 

Doing a simple whitelist will also block any users trying to load the page from a non-corporate IP, such as coffee shop, home, or mobile device. To slightly alleviate this risk, we can add a *RewriteCond* line to match requests from mobile user agents and allow those requests to access our site:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{HTTP_USER_AGENT}</span> <span style="color: mediumseagreen">"android|blackberry|googlebot-mobile|iemobile|ipad|iphone|ipod|opera mobile|palmos|webos"</span> <span style="color: mediumpurple">[NC,OR]</span><br>
RewriteCond <span style="color: dodgerblue">%{REMOTE_ADDR}</span> <span style="color: mediumseagreen">^100\.0\.0\.</span> <span style="color: mediumpurple">[OR]</span><br>
RewriteCond <span style="color: dodgerblue">%{REMOTE_ADDR}</span> <span style="color: mediumseagreen">^100\.0\.1\.</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REDIRECTION-URL.com/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
</div>

## Blacklisting

Blacklisting is inherently a reactive control, but can be useful on later phish campaigns after IR and control product's IPs have been identified. The ruleset is just flipping the last two lines so that the RewriteCond matches are redirected instead of proxied to the teamserver:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REMOTE_ADDR}</span> <span style="color: mediumseagreen">^100\.0\.0\.</span> <span style="color: mediumpurple">[OR]</span><br>
RewriteCond <span style="color: dodgerblue">%{REMOTE_ADDR}</span> <span style="color: mediumseagreen">^100\.0\.1\.</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REDIRECTION-URL.com/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span>
</div>

# Time-Based Filtering

To reduce the risk of off-hours IR loading our malicious site, we can use the time-based server variables built into mod_write. For example, if we wanted to limit users to only load our site during business hours:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{TIME_HOUR}%{TIME_MIN}</span> <span style="color: mediumseagreen">>0600</span> <span style="color: mediumpurple"></span><br>
RewriteCond <span style="color: dodgerblue">%{TIME_HOUR}%{TIME_MIN}</span> <span style="color: mediumseagreen"><2000</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://TEAMSERVER-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REDIRECTION-URL.com/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
</div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request time (hour and minute) </span> <span style="color: mediumseagreen">is greater than 0600 server local time, </span>AND<br>
<span style="color: dodgerblue">If the request time (hour and minute) </span> <span style="color: mediumseagreen">is less than 2000 server local time, </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span><span style="color: orange">from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
If the above conditions are not met, <span style="color: gold">change the entire request</span> <span style="color: orange"> to http://REDIRECTION-URL.com/ </span><span style="color: mediumvioletred">and drop any query strings from the original request. </span> <span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span>
</div>

Any requests received between the hours of 6am and 8pm are proxied to our malicious site. Any requests received outside that time range are redirected. Time calculation is based upon the server's locale settings. Use the `date` command to verify the time zone before configuring this ruleset.

We can also filter based on the day of the week with the *%{TIME_WDAY}* server variable. The regex uses 0 - 6 to represent Sunday through Saturday. To match days, use *<*, *>*, or *=* in the regex portion of the *RewriteCond* line. For example, the following *RewriteCond* rule matches Monday through Friday:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteCond <span style="color: dodgerblue">%{TIME_WDAY}</span> <span style="color: mediumseagreen">>0</span> <span style="color: mediumpurple"></span><br>
RewriteCond <span style="color: dodgerblue">%{TIME_WDAY}</span> <span style="color: mediumseagreen"><6</span> <span style="color: mediumpurple"></span><br>
</div>

# Summary

Apache mod_rewrite provides a wealth of powerful features we can use to make a phishing campaign more resilient to detection. We can filter requests based upon a number of properties, including user agent, source IP, and time. By redirecting or proxying those requests to an innocuous website we increase the chances that the investigator or security appliance will disregard the phish and continue allowing access. 

# Strengthen Your Phishing with Apache mod_rewrite Posts

* [Strengthen Your Phishing with Apache mod_rewrite and Mobile User Redirection]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/)
* [Invalid URI Redirection]({{site.baseurl}}/2016-03-29-invalid-uri-redirection-with-apache-mod_rewrite/)
* [Operating System Based Redirection with Apache mod_rewrite]({{site.baseurl}}/2016-04-05-operating-system-based-redirection-with-apache-mod_rewrite/)
* Combatting Incident Responders with Apache mod_rewrite
* [Expire Phishing Links with Apache RewriteMap]({{site.baseurl}}/2016-04-19-expire-phishing-links-with-apache-rewritemap/)


{:#resources}

# Resources

* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
	* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)
