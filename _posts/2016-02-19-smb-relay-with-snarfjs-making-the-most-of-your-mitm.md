---
layout: post
title: SMB Relay with Snarf
subtitle: Making the Most of Your MitM
summary: Leverage Snarf to maximize the value gained from your SMB Relaying, especially without Local Admin
tags:
- smb relay
---

SMB Relay is a well-known attack that involves intercepting SMB traffic and relaying the NTLM authentication handshakes to a target host. This post assumes you already understand the basics of SMB Relay (if not I *highly* suggest you check out Mark Baggett's SANS post [SMB Relay Demystified and NTLMv2 Pwnage with Python](https://pen-testing.sans.org/blog/2013/04/25/smb-relay-demystified-and-ntlmv2-pwnage-with-python)). SMB Relay has hands down been the most frequent foothold I've found on internal network pentests; however, sometimes the users in my broadcast domain don't seem to have Local Administrator rights on any of the targeted hosts or AV is making the process take a lot longer. This is where [Snarf](https://github.com/purpleteam/snarf) comes to the rescue.


I was on a network once where NBNS/LLMNR traffic was infrequent and administrator rights were limited. After a couple hours of running Responder with Impacket's [smbrelayx](https://github.com/CoreSecurity/impacket/blob/master/examples/smbrelayx.py), I had gotten nowhere on the pathway. None of the poisoned victims were administrators on the targeted hosts. Next I tried Snarf with Responder. Snarf provided NTLM handshakes for every poisoned user, allowed me to enumerate local administrative rights on a number of hosts, and provided me a full domain user list. From this point, I threw the handshakes in John, located a host with excessive administrative rights and ran a one or two guess online password brute force attack. This information eventually led to Domain Administrator rights and access to the crown jewels of the client's sensitive data. The pathway would have taken significantly longer, if it would have even been possible, without Snarf.


Snarf is a tool written in NodeJS by [Victor Mata](http://www.offense-in-depth.com/) and [Josh Stone](http://joshstone.us/) that keeps authenticated sessions alive and allows an attacker to run multiple commands through one successful relay. Snarf waits for the poisoned client to finish its transaction with the server (target), allows the client to disconnect from our host, and keeps the session between our host and the target alive. Now we can run tools through the hijacked session under the privilege of the poisoned user. 


For more info about the inner workings of the tool, check out Victor and Josh's NOLAcon '14 ([slides](http://www.josho.org/software/snarf-nolacon-presentation.pdf) , [demo](https://youtu.be/oBSrcrdRLyA)) and Derbycon 2014 ([video](https://youtu.be/X0S4Uuf3yUs)) presentations.
 

## Installation
Snarf requires nodejs and some attack or tool to get in the middle of an SMB connection. Spiderlabs' [Responder](https://github.com/SpiderLabs/Responder) is my personal favorite tool for this task - clone it as well if you don't already have it.

```plaintext
apt-get install nodejs
git clone https://github.com/purpleteam/snarf.git
```

## Usage
Make sure to start SnarfJS prior to any other tools being used for the SMB Relay attack; (such as Responder). This allows SnarfJS to bind to TCP port 445.

```bash
nodejs snarf.js <your-ip> -d <target-ip>
```

Snarf will output the following command to run to set an iptables rule:

```bash
iptables -t nat -A PREROUTING -p tcp --dport 445 -j SNARF
```

After running the iptables command, navigate in a browser to *127.0.0.1:4001*.

![Expiring a session](/assets/snarf/control.png)

In the top left, under Control, you can change the targeting mode from a single target to multiple targets. When this mode is enabled, SMB connections will be sent round robin at the targets in the list. This is a good method for determining local admin group members on hosts within your broadcast domain.

When a session comes in, it will initially be unavailable for interaction. This is because the victim is still communicating with the target. To make any active session available for selection, click Expire.

![Expiring a session](/assets/snarf/snarf-demo.gif)

You may notice the Hash column above. This is the MS-CHAP authentication handshake used to authenticate to the target. MS-CHAP handshakes cannot be passed like a full NTLM hash; however, we can try to crack the hash. If the authentication protocol is NTLMv1, you can try to use rcracki and rainbow tables to recover the plaintext password. See Foofus' [LM/NTLM Challenge/Response Authentication](http://h.foofus.net/?page_id=63) post for more details. Note that this attack will not work if the users' password is 15 or more characters and/or LM hashing is disabled. If the authentication protocol is NTLMv2, you can use john or oclhashcat to try and crack the password.


Now we have a working session that we can run tools through to investigate the target host and get code execution on if we have local admin rights.


## Tools

To use tools with Snarf, we will point the tool at localhost and enter any text as the username and password options. Snarf will listen for the connection, strip out the authentication details, redirect the tool to the target, and use the currently selected session's privileges. 

### rpcclient

*rpcclient* is a very useful tool in this attack, given the kinds of information that can be harvested. For more commands and ideas, check out these great posts:

* [more with rpcclient](http://carnal0wnage.attackresearch.com/2010/06/more-with-rpcclient.html) - Chris Gates
* [more of using rpcclient to find usernames](http://carnal0wnage.attackresearch.com/2007/08/more-of-using-rpcclient-to-find.html) - Chris Gates
* [Plundering Windows Account Info via Authenticated SMB Sessions](https://pen-testing.sans.org/blog/2013/07/24/plundering-windows-account-info-via-authenticated-smb-sessions) - Ed Skoudis

Useful Commands:

* lsaenumsid - get some SIDS for further enumeration
* lookupsids - turns SID into username
* queryuser RID - query user information
* getdompwinfo - enumerates password info
* enumdomusers - enumerates domain users
* enumalsgroups builtin - enumerates local groups 


**Enumerate Local Admins:**

```bash
enumalsgroups builtin
queryaliasmem builtin [RID] (ie queryaliasmem builtin 0x220)
lookupsids [SID]
```

**Enumerate Domain Users and Groups:**

(Based on a [tweet by Victor Mata](https://twitter.com/offenseindepth/status/551819778957266948))

*Replace the ##s with appropriate SID info*

```bash
for ((i=0;i<=1100;i++)); do rpcclient -U blah -N -c "lookupsids S-1-5-21-##-##-##-$i” 127.0.0.1 | tee -a domain-enum.txt; done
```

*NOTE: If you get timeout errors, throw `&& sleep 1` before `; done` to sleep for a second between attempts. *

**Enumerate Domain/Local Password Policy:**

```bash
rpcclient -W <workgroup|domain> -U user%pass localhost -c "getdompwinfo"
```

### net *

Use the base command of `net -U user%pass -I 127.0.0.1` with any of the following net commands

Useful commands:

* rpc group list - list local groups
* rpc group MEMBERS "Administrators" - list local admins 
* rpc user - list all users
* rpc user info <user> - list domain groups for specified user
* rpc user add <name> <password> [-F <user flags>] - add user
* rpc user delete <user> - delete user
* rpc share <host> - list shares on target host

*Note: Accessing any hidden shares, like C$, or user account changes require local administrative rights.*

### smbclient
See the [man page](https://www.samba.org/samba/docs/man/manpages/smbclient.1.html)

Connect to the target through Snarf:

```bash
smbclient \\\\127.0.0.1\\C$
```

All the normal Windows directory navigation works: `cd`, `dir`, and `ls`.

You can also upload/download files:


```plaintext
get <remote file name> [local file name]
put <local file name> [remote file name]
```

*Note: Accessing any hidden shares, like C$, require local administrative rights.*

### smbmap
[smbmap](https://github.com/ShawnDEvans/smbmap) is an SMB enumeration and interaction tool that can find weak share permissions and execute commands in a PSExec-like fashion. Note that this execution style writes to disk, so it may not fly on a Red Teaming engagement.

```bash
python /usr/share/smbmap/smbmap.py -H 127.0.0.1 -u user -p pass -x 'net group "domain admins" /domain'
```

*Note: Accessing the C$ share or executing commands through any hidden shares require local administrative rights.*

## Final Thoughts

Snarf is an awesome tool to use on internal pentest/red team engagements due to its poison once, exploit many abilities. This means a tester could stand up Snarf and Responder, poison one user, kill Responder, and enumerate a lot of information about the target domain with very minimal end-user impact. Personally, I feel like the next steps in using this tool are to determine, or create, a method to get remote code execution leveraging these tools without touching disk, allowing for usage on red team engagements.