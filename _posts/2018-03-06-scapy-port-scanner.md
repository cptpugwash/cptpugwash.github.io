---
layout: post
title:  "Scapy port scanner"
date:   2018-03-06
summary: Port scanner built using Scapy and python 
---

Port scanning is used to gather information about what networking services are running on a machine. There are several different ways in which this can be performed but in general it involves connecting to a port and listening for a response. From the response or lack of response we can infer if a port is open or closed. This can be done using Scapy to craft packets, send them, and parse the response. 

The response depends on the type of service and the protocol it uses, for example TCP is connection oriented whereas UDP is not. There is also the possibility that a firewall might perform an action on the packets (eg drop them), depending on how it is configured.

The following are a few different ways in which to determine if a network service is listening.

## SYN scan
This technique works by abusing how TCP establishes connections using the TCP handshake. It does this by trying to establish a TCP connection (sends SYN packet), if the machine replies with a reset (RST) then there's no service listening. However if it replies with SYN-ACK that means there is a service responding to the initial connection. 

We know a port is open because a TCP service will respond with the SYN-ACK flags to establish a handshake. If the port is closed the machine will send a RST packet. This is shown in figure 1.

![tcp handshake][image1]

In Scapy we can create a SYN packet by setting the TCP flag to "S", sending it and waiting for a response. Below is a simple SYN scan:

<figure class="lineno-container">
{% highlight python linenos=table %}
def syn_scan(target, ports):
	print("syn scan on, %s with ports %s" % (target, ports))
	sport = RandShort()
	for port in ports:
		# send a TCP packet with the syn flag set
		pkt = sr1(IP(dst=target)/TCP(sport=sport, dport=port, flags="S"), timeout=1, verbose=0)
		# checking response
		if pkt != None:
			# if it has a tcp layer, check the flags
			if pkt.haslayer(TCP):
				if pkt[TCP].flags == 20:
					print_ports(port, "Closed")
				elif pkt[TCP].flags == 18:
					print_ports(port, "Open")
				else:
					print_ports(port, "TCP packet resp / filtered")
			elif pkt.haslayer(ICMP):
				print_ports(port, "ICMP resp / filtered")
			else:
				print_ports(port, "Unknown resp")
				print(pkt.summary())
		else:
			print_ports(port, "Unanswered")
{% endhighlight %}
</figure>

The image below shows a SYN scan of ports 22 and 80, it also shows the TCP handshake captured in Wireshark. From the image we can see port 22 is closed and that port 80 is open because of the SYN-ACK packet being sent back.
![wireshark syn scan][image2]


## UDP scan
A UDP scan is more challenging to perform as the protocol is connectionless and doesn't perform a handshake to setup a connection. However when a port is closed the machine will typically send a ICMP error back "port unreachable". If no response is received then the port might be open or filtered by the firewall. Some services might respond with a UDP packet if open as well.

![udp handshake][image4]

In Scapy we create a UDP packet with the destination port, a timeout is also set as an open port might not send a reply.

<figure class="lineno-container">
{% highlight python linenos=table %}
def udp_scan(target, ports):
	print("udp scan on, %s with ports %s" % (target, ports))
	for port in ports:
		# send a UDP packet to a port
		pkt = sr1(IP(dst=target)/UDP(sport=port, dport=port), timeout=2, verbose=0)
		# check the response
		if pkt == None:
			print_ports(port, "Open / filtered")
		else:
			if pkt.haslayer(ICMP):
				print_ports(port, "Closed")
			elif pkt.haslayer(UDP):
				print_ports(port, "Open / filtered")
			else:
				print_ports(port, "Unknown")
				print(pkt.summary())
{% endhighlight %}
</figure>

Image below shows a UDP scan of ports 69 and 80, note there is no reply packet for port 69 whereas port 80 responds with ICMP port unreachable meaning it is closed.
![wireshark udp scan][image3]


## Xmas scan
An Xmas scan is a TCP scan that involves setting FIN, PSH and URG in one TCP packet (known as a Christmas tree packet). This method abuses the behaviour of RFC 793 compliment TCP systems. If a RST packet is received then the port is closed but if not then the port could be open or filtered. 

Note that Windows does not comply with the RFC and will send a RST packet regardless because of the malformed TCP packet, most Linux systems are and will behave according to the RFC.

Scapy command to create TCP xmas packet with the flags set. 

<figure class="lineno-container">
{% highlight python linenos=table %}
def xmas_scan(target, ports):
	print("Xmas scan on, %s with ports %s" %(target, ports))
	sport = RandShort()
	for port in ports:
		# send TCP packet, xmas flags are set
		pkt = sr1(IP(dst=target)/TCP(sport=sport, dport=port, flags="FPU"), timeout=1, verbose=0)
		# check response
		if pkt != None:
			if pkt.haslayer(TCP):
				if pkt[TCP].flags == 20:
					print_ports(port, "Closed")
				else:
					print_ports(port, "TCP flag %s" % pkt[TCP].flag)
			elif pkt.haslayer(ICMP):
				print_ports(port, "ICMP resp / filtered")
			else:
				print_ports(port, "Unknown resp")
				print(pkt.summary())
		else:
			print_ports(port, "Open / filtered")
{% endhighlight %}
</figure>

The results of a scan on ports 21, 22 and 80 show that port 21 and 80 sent no reply, whereas port 22 sent back a RST packet. From this we can infer that ports 21 and 80 are open and port 22 is closed.
![wireshark xmas scan][image5]


# Scapy port scanner
These functions can be combined into a python script to make a simple port scanner, note that all that's happening is creating a packet, sending it and listening for a response. The response or lack of response is then checked and parsed and is used to indicate if a port is open or closed.

The full script can be downloaded here: <https://github.com/cptpugwash/Scapy-port-scanner>

[image1]:../assets/images/scapy-port-scanner/tcp-handshake.png
[image2]:../assets/images/scapy-port-scanner/wireshark-syn-scan.png
[image3]:../assets/images/scapy-port-scanner/wireshark-udp-scan.png
[image4]:../assets/images/scapy-port-scanner/udp-handshake.png
[image5]:../assets/images/scapy-port-scanner/wireshark-xmas-scan.png