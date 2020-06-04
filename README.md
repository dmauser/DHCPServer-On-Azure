# DHCP-On-Azure
Proof of Cocentp on how to run Windows DHCP Servers on Azure

# Introduction

# OnPrem side

On my Home running Hyper-V with PFE Sense and two NICs (One NIC to my Home LAN and another to Hyper-V Internal). 
    Internal Hyper-V Network behind Pfsense I had W10 DHCP client.  
    Hyper-V Host installed RRAS and added DHCP Relay Agent listening on Internal Hyper-V Network. 
    PFSense with S2S VPN (BGP Enabled) to Azure VPN GW 

# Azure Side
•    Single VNET with Windows Server VM 
•    Azure VPN GW to terminate S2S VPN from OnPrem 

First I validated UDP traffic from my home DHCP Relay Agent hosted in Hyper-V would correctly arrive at Azure VM before spending time configuring DHCP Server side. It worked as expected based on what I explained on my last thread. Here is frame capture on Azure side (10.100.0.5) from the Relay Agent DISCOVERY request (192.168.2.10).
 
192.168.2.10    10.100.0.5    DHCP    DHCP:Request, MsgType = DISCOVER, TransactionID = 0xDE0A1442
Ipv4: Src = 192.168.2.10, Dest = 10.100.0.5, Next Protocol = UDP, Packet ID = 2100, Total IP Length = 328
Udp: SrcPort = BOOTP server(67), DstPort = BOOTP server(67), Length = 308
Dhcp: Request, MsgType = DISCOVER, TransactionID = 0xDE0A1442
 
    Relay Agent that Installed on my Hyper-V via RRAS represented above by source IP 192.168.2.10. 
    10.100.0.5 is Azure VM running Windows Server 2016 but no DHCP Server at this time. 

Main issue found: DHCP Server can be installed fine but it will not run on a Azure VM because it will not listed on interfaces that are using dynamic IP aka DHCP clients. Event Viewer shows:
Error    3/7/2020 1:00:29 PM    DHCP-Server    1041    None

The DHCP service is not servicing any DHCPv4 clients because none of the active network interfaces have statically configured IPv4 addresses, or there are no active interfaces.

Ok, we need a static IP on the VM and the only way to get that is adding a Loopback Adapter  . I added IP 10.100.0.100 on that NIC. No more error on DHCP after restart the service and just to make sure checked netstat and UDP 67 was correctly listening on 10.100.0.100.
 
 
Now I needed to get DHCP Relay Agent traffic to Loopback and most of you know the way to do that is setup up Load Balancer (ILB) in front of the VM with that Loopback IP and enable Direct Server Return on. Setup ILB with same Front End IP 10.100.0.100, add Azure VM as back end, setup Probe (to 3389) + LB Rule to UDP 67.
 
Configured DHCP Server my OnPrem DHCP Scope to offer range 192.168.2.100-192.168.1.100.
Here is the end result:
Full  DHCP DORA between Relay Agent and DHCP Server running on Azure VM:
  
 
DHCP Server shows lease:
 
  
 
End result on W10 side:
  

