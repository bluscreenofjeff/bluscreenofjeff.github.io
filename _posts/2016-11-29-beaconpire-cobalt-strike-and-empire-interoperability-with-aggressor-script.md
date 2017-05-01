---
layout: post
title: Beaconpire - Cobalt Strike and Empire Interoperability with Aggressor Script
summary: Controlling Empire listeners and passing sessions in Cobalt Strike with Aggressor script
tags: 
- cobalt strike
- aggressor 
- empire
image: /assets/beaconpire/beacon_to_empire_demo.gif
commentIssueId: 17
---


Tester flexibility and the ability to adapt to each environment’s unique controls and technologies is critical on assessments. Achieving an assessment’s objective often requires the use of multiple toolsets. Justin Warner ([@sixdub](https://twitter.com/sixdub)) wrote about the importance of tool diversity in his post [Empire & Tool Diversity: Integration is Key](https://www.sixdub.net/?p=627). Two toolsets I frequently use are Cobalt Strike and Empire. Sometimes, an assessment requires migrating from one toolset to another for a specific task or, worse, if incident responders block your primary toolset.  Currently, the most efficient way to pass sessions from one toolset to another is to use an existing session to run a payload for the target toolset. For example, passing a Beacon to Empire requires the tester to start an Empire listener, generate a launcher, and paste the code in each session to be passed. While this process isn’t inherently difficult, it is Manual session passing with Cobalt Strike and Empire is a tedious process andthat can interrupt a flow during post-exploitation, where efficiency is paramount. This post introduces beaconpire.cna - a script to help increase interoperability efficiency.


Beaconpire uses Beacon functionality and [Empire's RESTful API](https://github.com/adaptivethreat/Empire/wiki/RESTful-API) to manage Empire listeners and pass sessions between the toolsets, all from within Cobalt Strike. Beaconpire is written in Sleep and Aggressor Script and is entirely self-contained - no third party JSON scripts/libraries are required. Interacting with the RESTful API requires curl to be installed on the teamserver. Beaconpire currently supports the latest update in the *master* Empire branch. Empire 2.0 support is planned for the future. 

You can download the script [here](https://github.com/bluscreenofjeff/AggressorScripts/tree/master/Beaconpire).

A big thanks to Will Schroeder ([@harmj0y](https://twitter.com/harmj0y)) for his awesome [veil_evasion.cna](https://github.com/Veil-Framework/Veil-Evasion/blob/master/tools/cortana/veil_evasion.cna) script. I borrowed heavily from that script to make Beaconpire.


# Usage

To set up Beaconpire, load the beaconpire.cna script on your Cobalt Strike and start Empire's RESTful API server with `./empire --rest`. Copy the authentication token Empire outputs and paste it into the *Beaconpire -> Configure Server* menu in Cobalt Strike. Modify the server's IP and port to point to your Empire server and click Save.

![Server setup](/assets/beaconpire/server_setup.png)

## Empire Listener Management

Beaconpire can add and remove listeners on your Empire server. Open the listener management prompt with *Beaconpire -> Configure Empire Listeners*. The create listener prompt pulls current listener settings from Empire and presents a menu to modify settings as needed. Enter settings just as you would in an Empire prompt.

![Listener management demo](/assets/beaconpire/listener_management.gif)

## Beacon -> Empire Session Passing

Beaconpire provides a few options to pass Beacon sessions to Empire.

The first option is a batch session passing menu, accessed via the *Beaconpire -> Send Beacons to Empire* menu. The prompt presents a list of active Beacon sessions. Select one or multiple Beacons, click *Select Beacon(s)*, and select an Empire listener to which we will send the Beacon. Cobalt Strike pulls the one-line launcher stager, `usestager launcher`, from Empire and runs that PowerShell command in the selected session(s).

![Beacon to Empire session passing demo](/assets/beaconpire/beacon_to_empire_demo.gif)

The second option to send a Beacon session to Empire is by selecting one or more Beacon sessions in the sessions table, right click, and select *Send to Empire*. Cobalt Strike will prompt for an Empire listener selection, similar to the batch session passing menu, and run the launcher command in the selected session(s).

Beaconpire also provides the Beacon alias `sendtoempire` to quickly pass the current Beacon session to Empire. The alias does not require any parameters and will prompt for an Empire listener.

It should be noted that Beaconpire currently uses the Empire PowerShell launcher, `usestager launcher`, which presents OPSEC concerns in environments with commandline auditing or process auditing in place. 

![Beacon to Empire commandline auditing](/assets/beaconpire/powershell-cmd-line-auditing.png)

If you are operating in an environment with those controls in place, it's recommended that you avoid using Beaconpire's Beacon to Empire passing and opt to use a more covert manual method, such as [PowerPick](http://blog.cobaltstrike.com/2016/05/18/cobalt-strike-3-3-now-with-less-powershell-exe/) or [DLL Injection](http://blog.cobaltstrike.com/2016/06/08/session-passing-from-cobalt-strike/).

## Empire -> Beacon Session Passing

Pull sessions into Cobalt Strike from Empire with the *Beaconpire -> Pull in Empire Agents* menu. The prompt presents a list of active Empire agents. Select one or more of the agents, click *Select Agents* (or choose *Select ALL Agents*), and select a Beacon listener from the next prompt. Beaconpire will check Empire for an active foreign listener with settings matching the selected Beacon listener. If no Empire listener is present, Beaconpire will create the listener and run `invoke_shellcode` on the selected Agents.

![Empire to Beacon session passing demo](/assets/beaconpire/empire_to_beacon_demo.gif)

# Future

The following features are on the roadmap for future updates:

* [SSH Beacon](https://www.cobaltstrike.com/help-ssh) support (after [Empire 2.0](http://www.harmj0y.net/blog/empire/the-empire-strikes-back/) is merged into *master*)
* Alternate, more covert Beacon session passing methods
* Support username/password authentication to Empire

Beaconpire was designed to provide Cobalt Strike users basic interoperability with Empire. There are no plans to extend functionality beyond listener management and session passing. But, if you have a feature request or run into any bugs, feel free to hit me up!
