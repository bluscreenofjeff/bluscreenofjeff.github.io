---
layout: post
title: Expire Phishing Links with Apache RewriteMap
summary: Use Apache's RewriteMap to perform advanced HTTP request redirection, such as expiring phishing links and round-robin redirecting users to payloads.
tags: 
- mod_rewrite
- phishing
image: /assets/apache/expire-demo.gif
---

On more than a few occasions phishing recipients have forwarded my phish to IT. The first indication is usually when I'm watching the access logs like a hawk and see multiple GET requests with a user's token, yet haven't received any credentials or beacon sessions. Sometimes it turns out the user is being blocked by a technical control after the initial request, but other times we are told that the user did what their security training suggested and forwarded the email. Yay. 

I don't like the thought of being one email forward away from IR having free reign to peruse my phishing site in a sandbox. For a while I've wanted to limit this ability, allowing users to navigate to the malicious website or payload only once and sending all subsequent visitors to an innocuous website. After diving into [mod_rewrite]({{site.baseurl}}/2016-12-18-strengthen-phishing-with-apache-mod_rewrite/) recently, I found a method to accomplish this task natively in Apache with RewriteMap.


# Intro
Apache's RewriteMap module allows webmasters to map one string to another in a web request. For instance, if an online phone vendor changed their product pages from *product=12948* to *product=iphone-six-plus*, they could implement RewriteMap to handle the translation without having to edit every page's source. RewriteMap works by passing a key from the request to a text file, database, or program and returning the corresponding value.

We will use this functionality to pass a token from the request to a Python script that will validate an id parameter against a list of authorized ids and used ids. The script will return the id back to the Rewrite rules if the id is authorized and will return the string *nftoken* if the id is invalid. Then the *.htaccess* file will redirect the user appropriately.


![Phishing Link Expiration Demo](/assets/apache/expire-demo.gif)

In the demo above, notice that the requests with the *id=11111* parameter loads the requested resource once, but is redirected on the second attempt. The same occurs for the requests with the *id=22222* parameter.

# How it Works

The request processing consists of a *.htaccess* ruleset file, a list of authorized tokens, and a Python script for validating request tokens.

When the Apache server receives a request, the *.htaccess* ruleset is applied line by line from the top. The *.htaccess* file will assign the request's *id* parameter in the [query string]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/#rule-syntax) to a variable. The variable is passed to the Python script to determine if the contained token is valid and whether the token has been accessed more than the allowed number of times.

To validate the token, the Python script uses a line separated list of authorized tokens at the path `/var/expire/authusers.txt`. A valid request is handled using mod_proxy so the end-user never receives content directly from our Cobalt Strike teamserver, keeping our core infrastructure hidden. A request with an unauthorized or overused token will be redirected to the target company's website. The script writes used token counts to the file `/var/expire/used_ids.txt` in case we need to restart the server- this file and the *authusers.txt* file are parsed when the script launches and the Apache service is started. 

The Python script writes a log file to the path `/var/expire/process_log.txt` for real-time tracking and troubleshooting.


# Setup


## Configure RewriteMap Script

First we need to configure the apache2 server to run our script and send requests to it when the variable remap is used. It's important to note that the script will be started and run continuously when the apache2 service starts. If changes are made to the script you will need to restart the apache2 service for changes to take effect. Similarly, if the script fails and stops filtering requests properly (or times out after one successful request), it is likely that there is an issue with the script that should be troubleshot by manually starting the script or checking Apache's error logs.

Within the apache config (`/etc/apache2/apache.conf` on a default Debian install), add the following text in the [server context](https://httpd.apache.org/docs/2.4/mod/directive-dict.html#Context) of the config:

```plaintext
RewriteEngine on
RewriteMap remap prg:/var/expire/process.py
```

In order to use a *.htaccess* file to rewrite requests, we must tell apache to allow the files to override rules in the config. In the apache config, change **None** to **All** in the following block:

```plaintext
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```


Next, put the [Python script](https://github.com/bluscreenofjeff/Scripts/blob/master/Apache%20mod_rewrite/Link%20Expiration/process.py) at the path `/var/expire/process.py` on the server and change the ownership and permissions.

```bash
chown www-data:www-data /var/expire/process.py
chmod 744 /var/expire/process.py
```

By default, the script considers a token 'spent' after one use. To change this number, edit line 9 of the code.

**Note:** *If you are using Apache 2.2, you will need to configure a [RewriteLock](http://httpd.apache.org/docs/2.2/mod/mod_rewrite.html#rewritelock) setting to help prevent Apache or the script from hanging. Apache 2.4 handles this automatically.*

## Authorized Users

Get a list of tokens to allow through and place them in a line-separated file named `/var/expire/authusers.txt`. 

If using [Cobalt Strike](https://www.cobaltstrike.com/) for spearphishing, you can filter based on the *%TOKEN%* string that is appended to your phishing page's URL. 

To export the tokens from Cobalt Strike, go to Reporting -> Export Data. Exporting the tokens requires all phishes to be sent, so there will be some lag time between emails landing and filtering being fully implemented.

If you are not using Cobalt Strike, you could manually add tokens to URLs or leverage similar functionality in the tool you are using to spearphish.


## Create the *.htaccess* File

The actual filtering will be handled by a *.htaccess* file. Place the file at the root of the web server; it will apply to all subdirectories.

Within the *.htaccess file*, enter the following:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{HTTP_REFERER}</span> <span style="color: mediumseagreen">^http://phishdomain.com</span> <span style="color: tomato">[NC]</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-IP</span><span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: tomato">[P]</span><br>

RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">^id=(.*)</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">/$1?id=${remap:%1}</span> <span style="color: tomato">[R=302]</span><br>
RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">^id=nftoken</span> <span style="color: tomato">[OR]</span><br>
RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">!^id=*</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://COMPANY-DOMAIN/</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-IP</span><span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: tomato">[P]</span><br>
</div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's HTTP Referer</span> <span style="color: mediumseagreen"> starts with 'http://phishdomain.com' </span> <span style="color: tomato"> (ignoring case),</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path</span> <span style="color: orange">from the teamserver's IP, </span><span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
<span style="color: dodgerblue">If the request's query string</span> <span style="color: mediumseagreen">starts with 'id=', the text after 'id=' is assigned the variable name %1 and</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange"> to '/</span><span style="color: dodgerblue">original request path</span><span style="color: orange"> ?id=' and append the value returned by process.py.</span> <span style="color: tomato"> Redirect the user, changing the address bar, but continue evaluating rules.</span><br>
<span style="color: dodgerblue">If the quest's query string</span> <span style="color: mediumseagreen">starts with 'id=nftoken'</span> <span style="color: tomato"> OR</span><br>
<span style="color: dodgerblue">If the request's query string</span> <span style="color: mediumseagreen">does not start with 'id='</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange">to serve http://COMPANY-DOMAIN/</span> <span style="color: mediumvioletred">and drop any query_strings from original request.</span> <span style="color: tomato"> Do not evaluate further rules and redirect the user, changing their address bar.</span><br>
<span style="color: gold">Otherwise, change the entire request</span> <span style="color: orange">to serve the </span><span style="color:dodgerblue">original request path</span> <span style="color: orange">from the teamserver's IP </span><span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
</div>


You will need to modify *phishdomain.com*, *TEAMSERVER-IP*, and *COMPANY-DOMAIN* to fit your campaign.

This ruleset will allow any request referred by the spoofed domain, such as by a link. If you want to filter multiple pages, the first *RewriteCond* will require tweaking. The ruleset then assigns the variable *$1* to the string after *id=*, but only if that parameter is at the beginning of the Query String. The token (*%1*) is passed to the Python script for processing. If the token's access count is less than the configured allowed number, the token is returned, otherwise, the script returns the string *nftoken*. Any request with an id of *nftoken* or blank id is redirected to a real company page. Requests with authorized tokens are proxied to the Cobalt Strike teamserver. Since we are using Apache's proxy functionality, the user will never be served a resource directly from the Cobalt Strike teamserver. This should increase the lifetime of our core infrastructure. 


# Running the Script

Now restart the apache2 service. If the file `/var/expire/process_log.txt` was created, the script started. Any requests handled by the script will log to this file.

If you perform testing to make sure the filtering is working properly, just delete the file `/var/expire/used_ids.txt` and restart apache2 to clear the id blacklist. 

# Summary
Using Apache RewriteMap, we can apply extensive conditional filtering on requests, such as expiring links from phishing emails after they are visited. Expiring phishing links reduces the ability for incident responders to investigate our malicious website and payloads. Using the methodology presented in this blog post, we should be able to lengthen the effective operational time for our phishing campaign and slow the pace of a full investigation by incident responders.




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
