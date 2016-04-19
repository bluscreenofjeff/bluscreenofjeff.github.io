---
layout: post
title: Expire Phishing Links with Apache RewriteMap
summary: Use Apache's RewriteMap to perform advanced HTTP request redirection, such as expiring phishing links and round-robin redirecting users to payloads.
---

On more than a few occasions phishing recipients have forwarded my phish to IT. The first indication is usually when I'm watching the access logs like a hawk and see multiple GET requests with a user's token, yet haven't received any credentials or beacon sessions. Sometimes it turns out the user is being blocked by a technical control after the initial request, but other times we are told that the user did what their security training suggested and forwarded the email. Yay. 

I don't like the thought of being one email forward away from IR having free reign to peruse my phishing site in a sandbox. For a while I've wanted to limit this ability, allowing users to navigate to the malicious website or payload only once and sending all subsequent visitors to an innocuous website. After diving into [mod_rewrite]({{site.baseurl}}/2016-12-18-strengthen-phishing-with-apache-mod_rewrite/) recently, I found a method to accomplish this task natively in Apache with RewriteMap.


# Intro
Apache's RewriteMap module allows webmasters to map one string to another in a web request. For instance, if an online phone vendor changed their product pages from *product=12948* to *product=iphone-six-plus*, they could implement RewriteMap to handle the translation without having to edit every page's source. RewriteMap works by passing a key from the request to a text file, database, or program and returning the corresponding value.

We will use this functionality to pass a token from the request to a Python script that will validate an id parameter against a list of authorized ids and used ids. The script will return the id back to the Rewrite rules if the id is authorized and will return the string *nftoken* if the id is invalid. Then the *.htaccess* file will redirect the user appropriately.


![Phishing Link Expiration Demo](/assets/apache/expire-demo.gif)

In the demo above, notice that the requests with the *id=11111* parameter loads the requested resource once, but is redirected on the second attempt. The same occurs for the requests with the *id=22222* parameter.






# Strengthen Your Phishing with Apache mod_rewrite Posts

* [Strengthen Your Phishing with Apache mod_rewrite and Mobile User Redirection]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/)
* [Invalid URI Redirection]({{site.baseurl}}/2016-03-29-invalid-uri-redirection-with-apache-mod_rewrite/)
* [Operating System Based Redirection with Apache mod_rewrite]({{site.baseurl}}/2016-04-05-operating-system-based-redirection-with-apache-mod_rewrite/)
* [Combatting Incident Responders with Apache mod_rewrite]({{site.baseurl}}/2016-04-12-combatting-incident-responders-with-apache-mod_rewrite/)
* Expire Phishing Links with Apache RewriteMap


{:#resources}

# Resources

* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
	* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)
