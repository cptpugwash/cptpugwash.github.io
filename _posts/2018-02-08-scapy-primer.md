---
layout: post
title:  "Scapy primer"
date:   2018-02-08
summary: Primer on Scapy, a python packet manipulation framework used for networking and security testing
---

Scapy is a python packet manipulation framework used for creating and manipulating packets, this is useful for learning networking and security testing. It is recommended that you use Wireshark as well to visibly see what's happening on the network when packets are sent and received.

To install Scapy:
{% highlight python %}
pip install scapy
{% endhighlight %}
Once installed we can use the Scapy interpreter to play around before scripting. To start it just type `scapy`

Note that Linux might send reset (RST) packets when it receives packets sent using Scapy. This happens because the kernel receives the packets but didn't setup the connection so it sends a RST. To disable this behaviour use the command below, remember to put in your IP:
{% highlight python %}
iptables -A OUTPUT -p tcp --tcp-flags RST RST -s your_ip -j DROP	
iptables -L # Will show a list of rules set
{% endhighlight %}
### Basic commands for getting started
{% highlight python %}
conf - Displays the default config, items can be changed eg iface=mon0
lsc() - Lists commands available
ls() - Lists supported protocols
ls(TCP) - Passing a protocol will list default packet fields values
help(TCP) - Passing a command or protocol will display help about it
{% endhighlight %}
## Creating packets
Creating and modifying packets is pretty straight forward, we use the protocol name and pass it any values we want to set. Scapy will create the packet with default values eg ttl field values.

{% highlight python %}
# Lists packet fields and default values
ls(IP)
# Creates packet with default values
pkt = IP()
# Setting values
pkt = IP(dst="192.168.1.1")

# Methods used to show information about the packet
pkt.show()
	###[ IP ]### 
	  version= 4
	  ihl= None
	  tos= 0x0
	  len= None
	  id= 1
	  flags= 
	  frag= 0
	  ttl= 64
	  proto= hopopt
	  chksum= None
	  src= 192.168.1.108
	  dst= 192.168.1.1
	  \options\

pkt.summary()
	<bound method IP.summary of <IP  dst=192.168.1.1 |>>
{% endhighlight %}

We can create packets with  multiple layers using `/` to combine them, also note when sending packets Scapy will append the ethernet layer Ether() with the correct values so we don't always need to create it, unless we want to.

{% highlight python %}
# / is used to combine layers
pkt = IP(dst="192.168.1.1")/TCP(dport=80)
pkt.summary()
	'IP / TCP 192.168.1.108:ftp_data > 192.168.1.1:http S'

# Modifying packet data
pkt[TCP].dport = 21
{% endhighlight %}

#### Creating a ping packet
Here we use the commands IP() and ICMP() to create a ping packet, the fields will be filled with defaults. We can change them by setting them eg the IP dst, we need to set this in order to send our packet to an IP as it defaults to localhost. ICMP defaults are set to echo-request i.e ping request.

{% highlight python %}
# Creates ping packet with dst
pkt = IP(dst="192.168.1.1")/ICMP()

# Shows details of the packet
pkt.show()
	###[ IP ]### 
	  version= 4
	  ihl= None
	  tos= 0x0
	  len= None
	  id= 1
	  flags= 
	  frag= 0
	  ttl= 64
	  proto= icmp
	  chksum= None
	  src= 192.168.1.108
	  dst= 192.168.1.1
	  \options\
	###[ ICMP ]### 
	     type= echo-request
	     code= 0
	     chksum= None
	     id= 0x0
	     seq= 0x0

# Summary of packet
pkt.summary()
	'IP / ICMP 192.168.1.108 > 192.168.1.1 echo-request 0'
{% endhighlight %}

## Sending and receiving packets
There are a few ways to send and receive packets depending on what you want to do and which layer you want to send them on. Layer 3 is the network layer is routed based on the local table (usually the best method).

Layer 2 must include Ether() layer, will route based on it. 

Note that Scapy will populate some fields depending on the packet type and layer.
{% highlight python %}
# Sending packets
send() - Send a packet on layer 3  
sendp() - Send a packet on layer 2 

# Send our ping packet
send(pkt)
{% endhighlight %}
Scapy can also receive and listen for responses to sent packets.
{% highlight python %}
sr() - Send and receive packets layer 3  
srp() - Send and receive packets layer 2  

# Send and receive 1 packet layer 3 
sr1(pkt)
>>> sr1(pkt)
Begin emission:
.Finished to send 1 packets.
*
Received 2 packets, got 1 answers, remaining 0 packets
<IP  version=4L ihl=5L tos=0x0 len=28 id=29543 flags= frag=0L ttl=64 proto=icmp chksum=0x83bc src=192.168.1.1 dst=192.168.1.108 options=[] |<ICMP  type=echo-reply code=0 chksum=0xffff id=0x0 seq=0x0 |<Padding  load='\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00' |>>>

# Or we can send the packet in a loop
srloop(pkt, count=3)
{% endhighlight %}
Here we created a ping echo-request packet, send it and received an answer echo-reply. These are the same packets exchanged when pinging a machine from the command line.

When creating scripts with Scapy we will want to save the reply packets in a variable to work with, when sr() returns it returns two tuples (answered and unanswered packets) both of which are lists of tuples. The returned lists contain tuples which contain the sent and received (answered) packets.
{% highlight python %}
# Get responses to packet sent
ans, unans = sr(pkt)
>>> ans, unans = sr(pkt)
Begin emission:
.Finished to send 1 packets.
*
Received 2 packets, got 1 answers, remaining 0 packets

# Returned values stored in list of tuples, 1 in answered
>>> ans
<Results: TCP:0 UDP:0 ICMP:1 Other:0>
>>> unans
<Unanswered: TCP:0 UDP:0 ICMP:0 Other:0>

# Selecting the first tuple, it contains the sent and received packets
# ans = (sent, received) list of tuples for each packet sent and received
ans[0]
>>> ans[0]
(<IP  frag=0 proto=icmp dst=192.168.1.1 |<ICMP  |>>, <IP  version=4L ihl=5L tos=0x0 len=28 id=18439 flags= frag=0L ttl=64 proto=icmp chksum=0xaf1c src=192.168.1.1 dst=192.168.1.108 options=[] |<ICMP  type=echo-reply code=0 chksum=0xffff id=0x0 seq=0x0 |<Padding  load='\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00' |>>>)
# Selecting reply packet in tuple
ans[0][1]
# Get a summary of the packets
ans.summary()

# Typically used in scripts for working with received packets
for sent, received in ans:
	# Do stuff with received
	print(received.haslayer(ICMP))
{% endhighlight %}
## Sniffing packets
Rather than using Wireshark or tcpdump to capture packets we can use Scapy, this allows us to script and parse any packets of interest. sniff() is used to setup sniffing, we pass any configuration settings to it. Note that we can specify the iface, filter using BFP and pass any packets to a callback function 
{% highlight python %}
# Sniff on mon0, filter for icmp packets and pass them to parse_packet_func
sniff(iface="mon0", filter="icmp", prn=parse_packet_func, store=0)
{% endhighlight %}
Any packets matching the filter will be passed to a callback function (prn) where we can parse and manipulate the packets.
{% highlight python %}
# Any packets sniffed matching the filter will be passed to this function
def parse_packet_func(pkt):
	# Print summary of packet
	pkt.summary()
	# Check if it has the ICMP layer
	if pkt.haslayer(ICMP):
		# Print the type field of the ICMP packet
		print(pkt[ICMP].type)
{% endhighlight %}
All this function does it parse packets, checks if it has the ICMP layer and prints out the type of ICMP packet.

## Scripting
To use Scapy in python scripts you will have to import it using:
{% highlight python %}
from scapy.all import *
{% endhighlight %}

A couple of other good functions to be aware are:
{% highlight python %}
# Returns true if packet has the layer passed
pkt.haslayer(ICMP)
# Gets the layer passed
pkt.getlayer(ICMP)
{% endhighlight %}

### Example scripts
#### Sending a DNS request
To do this we need to create a DNS packet which is encapsulated in the lower layers. DNS uses UDP on port 53 for requests (this is set by default when creating UDP packets) then we just specify the DNS server in the IP layer. Also note that the DNS packet requires the qd field to be set using DNSQR with the domain request passed to it.

{% highlight python %}
# Import Scapy
from scapy.all import *
# Create DNS request packet
query = IP(dst="8.8.8.8")/UDP()/DNS(rd=1, qd=DNSQR(qname="www.google.co.uk"))
# Send DNS packet to DNS server
ans = sr1(query)

# Print the summary of the retuned packet
print(ans.summary())
ip = ans[DNS].an.rdata
{% endhighlight %}

#### Sending a HTTP GET request
Note there is no HTTP layer, we just send raw data (HTTP get request) and the HTTP server will parse it. But before we can send the request we need to establish a TCP connection using the handshake. This involves sending a syn packet and receiving a syn-ack from the server then sending an ack.

Once the handshake is complete we send the HTTP GET request, the server will send an ack followed by the HTTP header then the html content.

{% highlight python %}
from scapy.all import *

# Source port we are sending from
sport = 1337
# Get request in raw data
get = "GET / HTTP/1.1\r\nHost: 192.168.1.105\r\n\r\n"
# Create IP layer, dst = HTTP server
ip = IP(dst="192.168.1.105")

# Create syn packet to setup tcp handshake
syn = ip/TCP(sport=sport, dport=80, flags="S", seq=1337)
# Receive syn-ack
syn_ack = sr1(syn)

# Create ack packet
ack = ip/TCP(sport=sport, dport=80, flags="A", seq=syn_ack.ack, ack=syn_ack.seq+1)
send(ack)

# HTTP GET request
pkt = ip/TCP(sport=sport, dport=80, flags='PA', seq=syn_ack.ack, ack=syn_ack.seq+1)/get
ans = sr1(pkt)
# Response will be ack followed by the HTTP content
# Check wireshark for full content
print(ans.summary())
{% endhighlight %}