---
layout: post
title: Cobalt Strike OPSEC Profiles
summary: 
tags:
- aggressor
- cobalt strike
image: /assets/cobalt-strike-opsec-profiles/cobalt-strike-powershell-opsec-profile-demo.gif
commentIssueId: 29
---

Penetration tests and red team assessments often require operators to work multiple potential attack paths or perform multiple checks concurrently. Cultivating these myriad paths is what often leads operators to success in achieving their objectives. However, this execution method can also lead to an operator making a simple mistake, like running a "known bad" action for which there is a trivial detection. I can say I personally have been in the heat of an attack path and accidentally run PowerShell in an environment with very heavy PowerShell monitoring. It happens.

My coworker, Lee Christensen ([@tifkin_](https://twitter.com/tifkin_)), and I were recently discussing how to leverage automation to assist an operator in mundane tasks that can be neglected during such a test. Mundane tasks like maintaining strong situational awareness (identifying potential actions that could get you caught); critically important, but easy to forget when you're in the thick of it. Since we frequently operate with Cobalt Strike, Aggressor Script was where we naturally focused the discussion. We soon came up with a method of leveraging Aggressor Script to limit an operators ability to run commands that rely on known monitored methods in a given target environment, or as we termed them: OPSEC Profiles. Our profiles are heavily based upon the OPSEC considerations that Raphael Mudge ([@armitagehacker](https://twitter.com/armitagehacker)) published in his blog post [OPSEC Considerations for Beacon Commands](https://blog.cobaltstrike.com/2017/06/23/opsec-considerations-for-beacon-commands/).

The OPSEC Profiles are located on GitHub [here](https://github.com/bluscreenofjeff/AggressorScripts/tree/master/OPSEC%20Profiles). The GitHub repo currently includes profiles that limit commands that run cmd.exe, run powershell.exe, spawn new processes, rely on process injection, or create new services. The repo also includes a template file, allowing you to easily create a profile that limits commands being heavily monitored in your target environment.

The profiles are an Aggressor script that configures each Beacon command with a `block` or `enable` setting. *Block* disallows the command from running and displays an error, while *enable* allows the command to run unmodified. By selecting the appropriate commands, it's possible to configure Cobalt Strike to prevent most accidental command execution in target environments with mature detection capabilities.

# Usage

## Getting Started
To get started with a profile, select the profile that closest meets your needs and load the script as you would any other Aggressor Script. To load scripts, click *Cobalt Strike* in the top menu and select *Script Manager*. Clicking *Load* will pop up a file browser and allow you to graphically load the OPSEC Profile.

The profile immediately takes effect and will remain in effect until unloaded. Any command configured with the *block* setting will display an error message if it is run. It's important to note that the profiles limit command usage in the Beacon command console and the popup (right-click) menus in both the Beacon list and target list views. There are other GUI buttons, such as the Inject button within the Process Pane, that are not limited by the profiles. The OPSEC Profiles are intended to reduce the risk of operators running known detected commands, but caution should always be used when operating in a heavily-monitored environment. The profiles are not foolproof.

It's also important to note that OPSEC Profiles, and indeed Aggressor Scripts in general, only impact the client that loaded it. So, if you would like all of your operators to limit certain commands, they will all need to load the OPSEC Profile in their client.

## OPSEC Command
The OPSEC Profiles add a new command, `opsec`, to Cobalt Strike. This command will output a list of all modified commands and their corresponding block/enable setting.

For example:

```
beacon> opsec
[+] The current opsec profile has the following commands set to enable/block: 
[*] browserpivot - enable
[*] bypassuac - block
[*] cancel - enable
[*] cd - enable
[*] checkin - enable
[*] clear - enable
[*] covertvpn - block
[*] cp - enable
```

## Removing the Profiles

To remove the profiles, you can either:

* unload the script and load the [default.cna](https://www.cobaltstrike.com/aggressor-script/default.cna) script
* unload the script and restart the Cobalt Strike client

If you are running other scripts that modify Beacon's command registry or aliases, those scripts may need to be reloaded after the OPSEC Profile is unloaded.

# Available Profiles

The following profiles are available in the GitHub repo as of this post's writing:

* **cmd-execution.cna** - Prevents commands that rely on cmd.exe
* **powershell.cna** - Prevents commands that rely on powershell.exe
* **process-execution.cna** - Prevents commands that spawn a new process
* **process-injection.cna** - Prevents commands that use process injection
* **service-creation.cna** - Prevents commands that create new services
* **template.cna** - Includes all commands with an on/off variable for each command at the top of the script. Edit as needed for your specific use-case.

# Customizing Profiles

The **template.cna** profile is intended to provide an easily customizable base for your OPSEC Profile.

To customize the built-in commands, simply modify the enable/block value of the corresponding command. For example, the following code disables `bypassuac` and `covertvpn`:

```
%commands["browserpivot"] = "enable";
%commands["bypassuac"] = "block";
%commands["cancel"] = "enable";
%commands["cd"] = "enable";
%commands["checkin"] = "enable";
%commands["clear"] = "enable";
%commands["covertvpn"] = "block";
%commands["cp"] = "enable";
%commands["dcsync"] = "enable";
```

The OPSEC Profiles can block custom commands as well. Simply add a new line with the following text to the bottom of the existing list of commands in the profile:

```
%commands["COMMAND-NAME"] = "block";
```

The following example line blocks a command named "get-da":

```
%commands["get-da"] = "block";
```

To block custom GUI commands, you will need to modify the script further. Refer to lines [115 - 366](https://github.com/bluscreenofjeff/AggressorScripts/blob/master/OPSEC%20Profiles/template.cna#L115-L366) to see how the profiles prevent blocked built-in commands from running the corresponding Aggressor functions. This portion of the template profile is based on the *default.cna* script linked earlier in this post.

As an example, given the following custom popup menu code:

```
popup beacon_bottom {
    item 'Run "jobs"' {
        binput($1, "jobs");
        bjobs($1);
    }
}
```

The following code would block its execution given *jobs* was set to *block*:

```
popup beacon_bottom {
    item 'Run "jobs"' {
        if (%commands['jobs'] eq 'block') {	
            operror($1);			
        }
        else {         
            binput($1, "jobs");
            bjobs($1);
        }
    }
}
```

The modified code checks to see if `%commands['jobs']` is set to *block*, and, if so, executes the *operror* function. The *operror* function runs [openOrActivate](https://www.cobaltstrike.com/aggressor-script/functions.html#openOrActivate) and prints the error message to the Beacon log. 

For reference, here is the full *operror* function code:

```
sub operror {
	openOrActivate($1);
	berror($1,"This command's execution has been blocked. Remove the opsec profile to run the command.");
}
```

# Demo
Here's a demo of the PowerShell script in action:

{% include image.html file="/assets/cobalt-strike-opsec-profiles/cobalt-strike-powershell-opsec-profile-demo.gif" description="PowerShell OPSEC Profile Demo" %}

As you can see, before the script is loaded, PowerShell executes. Once the *powershell.cna* script is loaded, it displays an error when the operator attempts to run PowerShell. Since the profile executes before the command is passed to the Beacon itself, nothing is run on the compromised host.

# Summary

Penetration testers and red teamers often need to manage multiple attack paths at once during an assessment. While multitasking often means the tester can provide a more comprehensive assessment, it can also lead to an operator making mistakes. Such mistakes can include running known detected commands on a host in a highly-monitored target environment. Cobalt Strike OPSEC Profiles aim to reduce that risk by preventing operators from running "known bad" commands once the profile has been loaded. The profiles are fully customizable to meet your specific needs and can easily be extended to govern custom functionality provided by your own Aggressor scripts.

*This post and the OPSEC Profiles were co-written by Lee Christensen ([@tifkin_](https://twitter.com/tifkin_)) and me. For more from Lee, be sure to check him out on Twitter or on his [GitHub](https://github.com/leechristensen/).*

# Further Resources
* [OPSEC Considerations for Beacon Commands](https://blog.cobaltstrike.com/2017/06/23/opsec-considerations-for-beacon-commands/) - Raphael Mudge ([@armitagehacker](https://twitter.com/armitagehacker))
* [Aggressor Script Documentation](https://www.cobaltstrike.com/aggressor-script/index.html)