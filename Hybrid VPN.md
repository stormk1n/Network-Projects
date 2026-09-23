# VPN Overview

A Virtual Private Network provides a private tunnel to access resources securely over an unsafe network (mainly the internet)


**Client to site (C2S):** Remote user connects to the organization (similar to ovpn files used by THM and HTB labs).

**site to site (s2s):** Both branches of an organization route traffic through a secure tunnel.

<br>
<br>

# What we'll configure
A hybrid vpn model (both C2S and S2S in one) using IPSec protocol (openvpn is another vpn protocol, but not configured here)

## Prerequisites
1) IP Subnetting ([NetworkChuck - IP Addresses Playlist: 9vids = 2Hrs total](https://www.youtube.com/playlist?list=PLIhvC56v63IKrRHh3gvZZBAGvsvOhwrRF))
2) Knowlege on Access Control Lists (ACLs) ([Keith Barker - IP ACLs 18Mins](https://youtu.be/qt2-_RnjGh8))
3) Knowlege on VLANS would also proof usefull ([danscourses VLANS and Trunks - 9mins](https://youtu.be/aBOzFa6ioLw))
4) Understading of basic routing protocols (static routing) ([Paolo Reyes Static Routing - 10mins](https://youtu.be/z9e-6BATm0A))
5) Dynamic Ip addressing ([Ace Networker - Dynamic Vs Static Addressing 3mins](https://youtu.be/YMye-gjUiOI))
6) Basic understanding of server deployment (that is if going the extra mile is ok with you) [AD-DS](https://youtu.be/OtdOEiTozUE), [DNS](https://youtu.be/awRgw9ehmBI?t=178), [nginx](https://youtu.be/KonyowyG4pg), and [a sample of my configs](./Assets/Nephus/Nephus%20Labs%20Week%2001/04%20Server%20Config/) with [WDS](./Assets/winser12%20WDS%20config/)


## Tools Required
1) [David Bombal Getting started with gns3](https://youtu.be/Ibe3hgP8gCA?list=PLhfrWIlLOoKNFP_e5xcx5e2GDJIgk3ep6)
2) Gns3 (comes with wireshark, also needed in case it wasn't installed) [GNS3 Download](https://www.gns3.com/software/download)
3) Gns3 vm [GNS3VM Download](https://www.gns3.com/software/download-vm)
4) Available appliance files: [Router](https://github.com/GNS3/gns3-server/blob/master/gns3server/appliances/cisco-iosv.gns3a) and [switch](https://github.com/GNS3/gns3-server/blob/master/gns3server/appliances/cisco-iosvl2.gns3a) (Import the gns3 appliance files as templates or find a method that best suites you)
5) Basic usage of gns3 ([Once again, David Bombal Getting started with gns3](https://youtu.be/Ibe3hgP8gCA?list=PLhfrWIlLOoKNFP_e5xcx5e2GDJIgk3ep6))
6) VMs (used windows server 22, ubuntu and win7)
7) Cisco vpn client 5.0.07 which doesn't cost us and is compactible with win7 ([Download](https://www.superpit.com.au/vpn-software/vpnclient-winx64-msi-5-0-07-0290-k9/))
8) Hypervisor of choice

<br>
<br>

## Configuration Phases
Before we begin, heres what the final topology should look like
- Please see [note](#helpful-note) before anything else

<img src='./Assets/Hybrid VPN/Topology Overview.png' alt='Hybrid VPN Topology'>

<details><summary><h6 style='display:inline'>IP Addressing Plan</h6></summary>

| Site / Location | Department | Users | Subnet Address | CIDR | Subnet Mask | Usable IP Range |
| :--- | :--- | :---: | :--- | :---: | :--- | :--- |
| Branch | Sales | 120 | 192.168.20.0 | /25 | 255.255.255.128 | .1 – .126 |
| Branch | Production | 100 | 192.168.20.128 | /25 | 255.255.255.128 | .129 – .254 |
| HQ | IT | 40 | 192.168.10.0 | /26 | 255.255.255.192 | .1 – .62 |
| HQ | Admin | 20 | 192.168.10.64 | /27 | 255.255.255.224 | .65 – .94 |
| HQ | Server Farm | 2 | 192.168.10.96 | /29 | 255.255.255.248 | .97 – .102 |
| Remote | Remote Users | 40 | 192.168.30.0 | /26 | 255.255.255.192 | .1 – .62 |
| WAN Link | HQ ↔ Branch | 2 | 17.15.20.0 | /30 | 255.255.255.252 | .1 – .2 |
| WAN Link | Branch ↔ Rem. | 2 | 17.15.25.0 | /30 | 255.255.255.252 | .1 – .2 |

</details>

**Note:**
- The portable file ([Hybrid_VPN_Portable_File](./Portable%20Projects/Hybrid_VPN_Portable_File.gns3project)) doesn't come with the servers or remote client, as those vms are too large to upload
<br>
<br>

## Getting started
Now then, both C2S and S2S have been configured in 3 phases in this walkthrough

**Note**
- Due to router constraints, we are going to use IKEv1 and ISAKMP v1, but configurations for v2 can be found in the [Configs folder](./Assets/Hybrid%20VPN/Configs/)


## Phase 1
### S2S Phase1
In phase 1, we define the encryption method used between the two peers and how they authenticate each other
```
R2(config)#!! SITE-TO-SITE PHASE 1
R2(config)#
R2(config)#crypto isakmp policy 10
R2(config-isakmp)#encr aes 256
R2(config-isakmp)#hash sha256
R2(config-isakmp)#authentication pre-share
R2(config-isakmp)#group 14
R2(config-isakmp)#lifetime 86400
R2(config-isakmp)#exit
```

### S2S Phase2
Phase 2 takes our focus to configuring the actual tunnel
```
R1(config)#!! SITE-TO-SITE PHASE 2
R1(config)#!! 
R1(config)#crypto isakmp key pass123 address 17.15.20.2   ! <- peer for site-to-site, change the IP on the other router to match that of this router
R1(config)#crypto ipsec transform-set Sec esp-aes 256 esp-sha256
R1(cfg-crypto-trans)#mode tunnel
R1(cfg-crypto-trans)#exit
R1(config)#ip access-list extended Site2Site !! <- Here, we define the traffic we're intrested in protecting in our ACL
R1(config-ext-nacl)#permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
R1(config-ext-nacl)#exit
```


## S2S Phase3
Not standard practice, created a 3rd step for ease of my own work.
But here, we bind our configured VPN tunnel to the interface facing our peer
```
R1(config)#!! SITE-TO-SITE PHASE 3
R1(config)#!!
R1(config)#crypto map Site2Site 10 ipsec-isakmp
R1(config-crypto-map)#set peer 17.15.20.2
R1(config-crypto-map)#set transform-set Sec
R1(config-crypto-map)#match address Site2Site
R1(config-crypto-map)#exit
R1(config)#interface gi0/0
R1(config-if)#crypto map Site2Site
R1(config-if)#exit
```

### C2S Phase1
Phase one here is pretty much the same. We define our peers and encryption used
```
R2(config)#!! CLIENT VPN - IKEv1
R2(config)#
R2(config)#!! PHASE 1: IKEv1 & CLIENT IDENTITY
R2(config)# 
R2(config)# crypto isakmp policy 5
R2(config-isakmp)# encr aes 256
R2(config-isakmp)# hash sha
R2(config-isakmp)# authentication pre-share
R2(config-isakmp)# group 2
R2(config-isakmp)# lifetime 86400
R2(config-isakmp)# exit
R2(config)# 
R2(config)#!! CLIENT VPN POOL CONFIG + CLIENT IDTY
R2(config)#ip local pool ClientVPNPool 192.168.30.1 192.168.30.40  ! IPs for VPN clients, assigns dynamically
R2(config)#username Remoteusers password ClientP@ss123  ! <- simple local user
R2(config)#
```
Now, we map our client profile to the authentication group
```
R2(config)# crypto isakmp client configuration group Client_Group
R2(config-isakmp-group)# key ClientP@ss123
R2(config-isakmp-group)# pool ClientVpnPool
R2(config-isakmp-group)# acl Client2Site <- Access list
R2(config-isakmp-group)# exit
R2(config)# 

R2(config)# crypto isakmp profile Client_Group
R2(conf-isa-prof)# match identity group Client_Group
R2(conf-isa-prof)# client authentication list pre-share
R2(conf-isa-prof)# isakmp authorization list Client2Site
R2(conf-isa-prof)# exit
R2(config)# 
```

### C2S Phase2
Now, in our phase2, we define our client ACL and establish a tunnel on our VPN router
```
R2(config)# !! PHASE 2
R2(config)# ip access-list standard Client2Site
R2(config-ext-nacl)# permit ip 192.168.30.0 0.0.0.63 192.168.10.0 0.0.0.255
R2(config-ext-nacl)# permit ip 192.168.30.0 0.0.0.63 192.168.20.0 0.0.0.255
R2(config-ext-nacl)# exit
R2(config)# 
R2(config)# crypto ipsec transform-set Sec esp-aes 256 esp-sha-hmac
R2(cfg-crypto-trans)# mode tunnel
R2(cfg-crypto-trans)# exit
R2(config)# 
```

### C2S Phase3
Once our tunnel is established, we now bind it to our desired interface.
Once again, this (Phase3) isn't standard practice, may only be seen here
```
R2(config)# !! PHASE 3
R2(config)# crypto dynamic-map ClientGroup 10
R2(cfg-crypto-map)# set transform-set Sec
R2(cfg-crypto-map)# set isakmp-profile Client_Group
R2(cfg-crypto-map)# exit
R2(config)# 
R2(config)# crypto map Client2Site 10 ipsec-isakmp dynamic ClientGroup
R2(config)# 
R2(config)# int gi0/2
R2(config-if)# crypto map Client2Site
```

Final ACL configuration <br>
<img src='./Assets/Hybrid%20VPN/ACL.png' alt="Final ACL">
<br>

### Setting up or VPN Client
<img src='./Assets/Hybrid VPN/Cisco_Client_Config.png' alt='VPN Client'>

Once our configurations have been done, we save the file and test the results

<br>
<br>

# Results

## Veryfing S2S Connectivity
1) Verifying phase1 and phase2 with the command (on any of the routers)
```
R1(config)# do show crypto isakmp sa
```
Should return <br>
<img src='./Assets/Hybrid VPN/Isakmp SA.png' alt='Phase1 and phase2 results'>
<br>Our security association (SA) shows up as active.

**Note:**<br>
- We can't verify phase3 since it isn't a standard phase, so all we do next is to ping and capture traffic of a ping request and study how it goes
- Taking out the clouds nodes so wireshark doesn't capture traffic from our local network (caused the GNS3 app to crash out in my case and lead to vmnets shutting down after each crash)
<br>
<br>

2) Analyzing wireshark capture for presence of ESP packets
<img src='./Assets/Hybrid VPN/Site_WAN_ping.png' alt='S2S Capture' style='width: 100%; height: 100%'>
<br>The presence of the ESP packets indicate that our tunnel was successfully setup

## Veryfing C2S Connectivity

1) Confirming IP leasing with the command
```
R2(config)# do show ip local pool
```
Before and after our client connects.

We can confirm client access from

<img src='./Assets/Hybrid VPN/Ip_local_Pool.png'>
<br>which shows us the state of the local pool before and after our client connected

2) Once again, we analyze our traffic with wireshark to make sure we safe

<img src='./Assets/Hybrid VPN/ClientTrafficAnanlysis.png'>

<br>

# Conclusion
With results from both captures showing as ESPs, it confirms our VPN is up and running

<br>

# Troubleshooting Tips
1) GNS3 had a tendency of shutting down the vmnet interfaces, which caused a lot of trouble in verifying our remote clients connectivity.

2) The commands are strenuous and one miss-typed input could lead to troubleshooting for days, caused by a simple typo

3) Verify the ACL: for some reason, standard ACL worked best for C2S while extended worked great for S2S, (had to learn that the hard way)

<br>


## Helpful Note

There's a whole lot put in this network, we only focused on the VPN configuration, but this network had;
1) IP subnetting (which is seeing above)
2) Vlan creation
3) Routing protocols
4) Intervlan routing (router on a stick)
5) DHCP to automatically assign IP (configured on the router, not on the windows server)
6) ACL configuration (mentioned but not gone deep into)
7) AD-DS
8)  DNS (forward and reverse lookup zones)
9) Hosting a web site using nginx (custom made [lahfen-autos](https://github.com/stormk1n/Lahfen-Autos))
10) Configuring UFW on the ubuntu server
11) And finally our VPN which is a 2 in 1 setup (so there should be 12)
