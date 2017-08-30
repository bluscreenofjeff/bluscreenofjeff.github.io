---
layout: post
title: Randomized Malleable C2 Profiles Made Easy
summary: 
tags:
- cobalt strike
- malleable c2
- red team infrastructure
image: /assets/malleable-c2-randomizer/malleable-c2-randomizer-demo.gif
commentIssueId: 26
---

Malleable Command and Control (C2) profiles provide red teamers and penetration testers with a [wealth of options](https://www.cobaltstrike.com/help-malleable-c2) to modify how Cobalt Strike both appears on the wire and on the compromised host. Malleable C2 can be used to impersonate [actual threat actors](https://github.com/rsmudge/Malleable-C2-Profiles/tree/master/APT) or [normal web traffic](https://github.com/rsmudge/Malleable-C2-Profiles/tree/master/normal). As with every advancement in offensive tradecraft, blue teams and defensive products are bound to implement static signature-based protections. In my opinion, a defender should use any resources available, including signature-based detections; however, they shouldn't rely on any one defensive technique. As red teamers, it's our job to exercise the blue team's controls and processes and this is precisely what Malleable C2 profiles permit us to exercise.

In this blog post, I'll detail a script I wrote to randomize Malleable C2 profiles, allowing us to customize the same profile template every time we use it and hopefully reduce the chances of flagging static, signature-based detection controls. The script is available [here](https://github.com/bluscreenofjeff/Malleable-C2-Randomizer).

# The Script
The script randomizes Cobalt Strike Malleable C2 profiles through the use of a metalanguage, replacing keywords with random, pre-configured strings. In short, the script parses the provided template, substitutes the variables for a random value from either a provided or built-in wordlist, tests the new template with c2lint, and (if there are no c2lint errors) outputs the new Malleable C2 profile.

I've provided sample Malleable C2 profiles that are compatible with the script in the [Sample Templates](https://github.com/bluscreenofjeff/Malleable-C2-Randomizer/tree/master/Sample%20Templates) directory of the script repo.

Here is a sample of a profile that has been modified to use the script's metalanguage, followed by a resulting profile:

```plaintext
# This profile has been modified to use with the Malleable C2 Profile Randomizer

#
# Amazon browsing traffic profile
# 
# Author: @harmj0y
#

set sleeptime "%%number:2%%00";
set jitter    "1%%number%%";
set maxdns    "24%%number%%";
set useragent "%%useragent%%";

http-get {

    set uri "/s/ref=nb_sb_noss_1/%%number:3%%-%%number:8%%-%%number:7%%/field-keywords=%%word%%";

    client {

        header "Accept" "*/*";
        header "Host" "www.amazon.com";

        metadata {
            base64;
            prepend "session-token=";
            prepend "skin=noskin;";
            append "csm-hit=s-%%alphanumeric:20%%|%%number:13%%";
            header "Cookie";
        }
    }
}
```

```plaintext
# This profile has been modified to use with the Malleable C2 Profile Randomizer

#
# Amazon browsing traffic profile
# 
# Author: @harmj0y
#

set sleeptime "7400";
set jitter    "19";
set maxdns    "246";
set useragent "Mozilla/5.0 (Windows NT 10.0; WOW64)";

http-get {

    set uri "/s/ref=nb_sb_noss_1/684-10075672-1686806/field-keywords=year";

    client {

        header "Accept" "*/*";
        header "Host" "www.amazon.com";

        metadata {
            base64;
            prepend "session-token=";
            prepend "skin=noskin;";
            append "csm-hit=s-ub5oBmGd0pnDoImCjDyK|2539750656854";
            header "Cookie";
        }
    }
}
```

Porting existing profiles is straightforward, as shown with the [Amazon profile](https://github.com/rsmudge/Malleable-C2-Profiles/blob/master/normal/amazon.profile) above. To start, replace or add any global settings with the appropriate script variables, such as *useragent* and *spawnto*. It's worth the time to build as many of these options into the profile template as possible, as they will vary the traffic from the stock profile even more. Next, replace any long or seemingly random strings with the appropriate metalanguage characterset, such as those on line 16 above.

# Usage
The script requires a modified Malleable C2 profile as input with `-profile` (or `-p`). If you intend to have *c2lint* test the profile before saving (recommended), either run the script from your Cobalt Strike directory or provide the Cobalt Strike directory with the `-cobalt` (`-d`) parameter. All other parameters are optional and will use built-in lists if a custom list is not provided.

Full usage options:
```python
python malleable-c2-randomizer.py [-h] -profile PROFILE
                                  [-count COUNT]
                                  [-cobalt COBALT]
                                  [-output OUTPUT]
                                  [-notest]
                                  [-charset CHARSET]
                                  [-wordlist WORDLIST]
                                  [-useragent USERAGENT]
                                  [-spawnto SPAWNTO]
                                  [-pipename PIPENAME]
                                  [-pipename_stager PIPENAME_STAGER]
                                  [-dns_stager_subhost DNS_STAGER_SUBHOST]
```

## Basic Options

| Parameter | Description |
| ----- | ----- |
| -profile, -p | Path to the Malleable C2 template to randomize (REQUIRED) |
| -count, -c | The number of randomized profiles to create {Default = 1} |
| -cobalt, -d | The directory where Cobalt Strike is located (for c2lint) {Default = current directory} |
| -output, -o | Output base name {Default = template basename and random string} |
| -notest, -n | Skip testing with c2lint (Flag) |

## Custom Wordlists
If no wordlist is provided, a built-in list will be used by default. For more information about creating these lists, see [below](#building-wordlists) or the [Sample Wordlists](https://github.com/bluscreenofjeff/Malleable-C2-Randomizer/tree/master/Sample%20Lists) folder in the script repo.

| Parameter | Description |
| ----- | ----- |
| -charset | File with a custom characterset to use with the %%customchar%% variable |
| -wordlist | File with a list of custom words to use with the %%word%% variable |
| -useragent | File with a list of useragents |
| -spawnto | File with a list of custom spawnto processes |
| -pipename | File with a list of custome pipenames |
| -pipename_stager | File with a list of custom pipename_stagers |
| -dns_stager_subhost | File with a list of custom dns_stager_subhosts |


Most of these wordlist variables are directly related to Malleable C2 options. For more information about what these profile options do, check out the [official documentation](https://www.cobaltstrike.com/help-malleable-c2).

## Substitution Metalanguage
The substitution metalanguage comprises specific variables, some of which allow optional repetition counts, wrapped in double percentage signs, like so:

```
%%variable:count%%
```

As another example, the following variable will result in 20 alphanumeric characters:

```
%%alphanumeric:20%%
```

## List of Variables

| Variable | Description | Supports Count? |
| ----- | ----- | ----- |
| alphanumeric | Outputs a random mixed-case ascii letter or digit | Yes |
| alphanumspecial | Outputs a random mixed-case ascii letter, digit, or punctuation | Yes |
| alphanumspecialurl | Outputs a random mixed-case ascii letter, digit, or one of the following characters: `-._~` | Yes |
| alphaupper | Outputs a random uppercase ascii letter | Yes |
| alphalower | Outputs a random lowercase ascii letter | Yes |
| alpha | Outputs a random ascii letter | Yes |
| number | Outputs a random digit | Yes |
| hex | Outputs a random hexadecimal digit | Yes |
| netbios | Outputs a random mixed-case ascii letter, digit, or one of the following characters: `!@#$%^&)(.-'_{}~` | Yes |
| custom | Maps to a random character in the provided *charset* file | Yes |
| word | Outputs a random word from the provided or built-in wordlist | Yes |
| useragent | Outputs a random useragent from the provided or built-in list | No |
| spawnto_x86 | Outputs a random x86 process path from the provided or built-in list | No |
| spawnto_x64 | Outputs a random x64 process path from the provided or built-in list | No |
| pipename | Outputs a random pipename from the provided or built-in list | No |
| pipename_stager | Outputs a random pipename_stager from the provided or built-in list | No |
| dns_stager_subhost | Outputs a random dns_stager_subhost from the provided or built-in list | No |

## Building Wordlists
Wordlist files are simply line-separated, tab-separated, or continuous strings (depending on the wordlist type) place in a text file.

The following wordlists should be line-separated with each entry on a new line:

* wordlist
* useragent
* pipename
* pipename_stager
* dns_stager_subhost

The spawnto wordlist is a bit more complicated. Malleable C2 requires an x86 and x64 option to modify all process spawning. Therefore, each line of the wordlist should contain both the x86 and x64 process paths separated by a tab, with the x86 process listed first. For example:

```
%windir%\\syswow64\\eventvwr.exe	%windir%\\sysnative\\eventvwr.exe
```

It's important to note that the `syswow64` and `sysnative` strings in the process paths should be **lowercase**.

The final wordlist type is the custom characterset. This file should include any characters for the script to randomly substitute. For example, a charset file of `AEIOUY` and a variable of `%%custom:5%%` will output five random characters from the charset string. When building this characterset, bear in mind that some characters are prohibited from appearing in a URI and may interfere with Beacon's communications.

For sample wordlists, see the [Sample Wordlists](https://github.com/bluscreenofjeff/Malleable-C2-Randomizer/tree/master/Sample%20Lists) directory in the script repo.

# Demo

{% include image.html file="/assets/malleable-c2-randomizer/malleable-c2-randomizer-demo.gif" description="Malleable C2 Randomizer Script Demo" %}

# Summary
Malleable C2 provides pentesters and red teamers with the valuable ability to modify how traffic appears on the wire and alter some endpoint signatures. A wealth of profiles are available on Raphael Mudge's [GitHub repo](https://github.com/rsmudge/Malleable-C2-Profiles), which is great to get up-and-running; however, these profiles have likely already been signatured by blue teams or defensive products. The [Malleable C2 randomizer script](https://github.com/bluscreenofjeff/Malleable-C2-Randomizer) covered in this post provides a mechanism to easily use a unique profile on each teamserver.

# Further Resources
* [Malleable Command and Control Documentation - Raphael Mudge](https://www.cobaltstrike.com/help-malleable-c2)
* [Malleable C2 Profiles - GitHub](https://github.com/rsmudge/Malleable-C2-Profiles)
* [How to Write Malleable C2 Profiles for Cobalt Strike - Jeff Dimmock](https://bluescreenofjeff.com/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/)
* [Cobalt Strike 2.0 - Malleable Command and Control - Raphael Mudge](http://blog.cobaltstrike.com/2014/07/16/malleable-command-and-control/)
* [Cobalt Strike 3.6 - A Path for Privilege Escalation - Raphael Mudge](http://blog.cobaltstrike.com/2016/12/08/cobalt-strike-3-6-a-path-for-privilege-escalation/)
* [A Brave New World: Malleable C2 - Will Schroeder](http://www.harmj0y.net/blog/redteaming/a-brave-new-world-malleable-c2/)