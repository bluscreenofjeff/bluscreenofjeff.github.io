---
layout: post
title: How to Make Communication Profiles for Empire
summary: How to create communication profiles to shape Empire C2 traffic.
tags:
- empire
featuredimage: /assets/empire-communication-profiles/default-profile-packet-capture.png
---

In a [recent post]({{site.baseurl}}/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/), I detailed how to make a Malleable C2 profile for Cobalt Strike. Malleable C2 profiles provide an operator with the ability to shape how defenders will see, and potentially categorize, C2 traffic on the wire. Communication Profiles in Empire provide similar functionality. This increases our chances of evading detection, allows us to emulate specific adversaries, or masquerade as widely-used applications on our target's network.

# Empire Communication Profiles

With Communication Profiles, we can customize options for Empire's GET request URIs, user agent, and headers. A basic profile consists of each element, separated by the pipe character, like this:

```plaintext
GET request URI | User Agent | Header #1
```

Here is a sample profile for [Comfoo](https://github.com/EmpireProject/Empire/blob/master/data/profiles/comfoo.txt):

```plaintext
/CWoNaJLBo/VTNeWw11212/|Mozilla/4.0 (compatible; MSIE 6.0;Windows NT 5.1)|Accept:image/gif, image/x-xbitmap, image/jpeg, image/pjpeg, */*|Accept-Language:en-en
```

Profiles can incorporate multiple request URIs and Headers by separating URIs with commas and separating additional Headers with pipes, like this:

```plaintext
GET request URI #1,GET request URI #2 ... | User Agent | Header #1 | Header #2 ...
```

The default profile offers an example of this in action:

```plaintext
/admin/get.php,/news.asp,/login/process.jsp|Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko
```

To configure a custom Communication Profile, we have two options:

1) We can configure the text of a profile directly in the Empire console under the Listener Options with the command `set DefaultProfile`.

![Configuring a Communication Profile](/assets/empire-communication-profiles/configure-profile.png)

2) We can modify the file [/setup/setup_database.py](https://github.com/EmpireProject/Empire/blob/293f06437520f4747e82e4486938b1a9074d3d51/setup/setup_database.py#L50) before Empire's initial setup to change the default profile.

Once the setting is configured, you launch the listener normally and we're set.

![Default Communication Profile Packet Capture](/assets/empire-communication-profiles/default-profile-packet-capture.png)

>Sidebar: Communication Profiles provide the ability to perform [Domain Fronting](http://www.icir.org/vern/papers/meek-PETS-2015.pdf). For details on how to set this up, see Chris Ross's ([@xorrior](https://twitter.com/xorrior)) post [Empire Domain Fronting](https://www.xorrior.com/Empire-Domain-Fronting/).

# Porting Malleable C2 Profiles

Since Communication Profiles and Malleable C2 profiles are both modifying the same traffic elements, it's very easy to port Malleable C2 profiles to Communication Profiles.

For illustration, we'll port the Malleable C2 [bingsearch_getonly](https://github.com/bluscreenofjeff/MalleableC2Profiles/blob/master/bingsearch_getonly.profile) profile. Here is a relevant snippet of the profile that structures the client's communications:

```plaintext
set useragent "Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko";

---snipped---

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
```

In short, the profile:

* defines the user agent
* sets the base URI to `/search/`
* base64 encodes host metadata in a URL-friendly format and assigns the output to the URL parameter `q`
* appends the parameters/query strings `go=Search&qs=form&QBRE`
* adds a "Host", "Accept", and "Cookie" header

To port the profile over, we simply copy the profile elements, assemble them the into their end result, and arrange them in the format Empire expects:

```plaintext
/search?q=canary&go=Search&qs=bs&form=QBRE|Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko|Accept:text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8|Cookie:DUP=Q=GpO1nJpMnam4UllEfmeMdg2&T=283767088&A=1&IG
```

Since Empire doesn't provide a customization option for modifying the placements of the C2 data payload, we can assign an arbitrary value to the `q` parameter.

Here's how a request looks on the wire:

![Sample Bing Profile Request](/assets/empire-communication-profiles/sample-request.png)

And here's how the session looks:

![Bing Profile Session](/assets/empire-communication-profiles/sample-traffic.png)

As you can see, the individual requests look good, but looking at a whole session reveals an issue. It looks very odd that this user is continuously requesting the same search query on Bing.

Luckily, we can use Empire's ability to use random request URIs to vary the traffic a bit more:

```plaintext
/search?q=news&go=Search&qs=bs&form=QBRE,/search?q=weather&go=Search&qs=bs&form=QBRE,/search?q=movie%20tickets&go=Search&qs=bs&form=QBRE,/search?q=unit%20conversion&go=Search&qs=bs&form=QBRE|Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko|Accept:text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8|Cookie:DUP=Q=GpO1nJpMnam4UllEfmeMdg2&T=283767088&A=1&IG
```

That ends up looking like this:

![Multiple Request URIs in a Communication Profile](/assets/empire-communication-profiles/multiple-request-uris.png)

Now we've ported over a Malleable C2 profile and spruced it up a bit to help the traffic blend in a bit more.

If you get stuck while trying to port a profile over, take a look at the [official Malleable C2 documentation](https://www.cobaltstrike.com/help-malleable-c2) or my post [How to Write Malleable C2 Profiles for Cobalt Strike]({{site.baseurl}}/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/) for more syntax or function details.

# Summary

Communication Profiles provide Empire with customization options for modifying how its C2 traffic appears on the wire. The flexibility the profiles provide can prove highly valuable in trying to evade incident responders by blending into your target's baseline traffic. Additionally, it provides the ability to emulate specific adversaries for training exercises and Blue Team tool honing.
