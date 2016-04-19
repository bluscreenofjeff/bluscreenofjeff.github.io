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

