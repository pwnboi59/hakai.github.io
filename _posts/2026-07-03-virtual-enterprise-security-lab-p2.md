---
title: "Virtual Enterprise Security Lab - Part 2: Opnsense Firewall Installation"
date: 2026-07-03
categories: [Cybersecurity, Home Lab]
---

## System Requirement
### 1. OPNsense
OPNsense is an open-source firewall and routing operating system based on FreeBSD. It's designed to protect networks, manage network traffic, and provide a wide range of security features for home networks, enterprise environments, and virtual labs. [Download in here](https://opnsense.org/download/)
![alt text](/assets/media/2026-07-02-virtual-enterprise-security-lab-p2/download_opnsense.png)

### 2. Bzip2
Using bzip2 to extract a downloaded .bz2 file. [Download in here](https://gnuwin32.sourceforge.net/packages/bzip2.htm)
```
C:\Program Files (x86)\GnuWin32\bin>bunzip2.exe "D:\LAB-CYBER\OPNsense-26.1.6-dvd-amd64.iso.bz2"
```
## How to configure OPNsense in VMware
### Step 1: Create the Virtual Machine
We will configure the VM to act as a central router/firewall for our lab network.
#### 1. VM Configuration
Create a new Virtual Machine in VMware Workstation. Select "Typical" and browse to your extracted OPNsense ISO file.
+ OS Type: FreeBSD 14 64-bit.
+ Disk: 20GB single file is sufficient.
+ Memory: Increase to 4GB.
+ Processors: 2 cores.

#### 2. Network Adapters (Crucial Step)
Before configuring the lab, we need to analyze the network diagram below.
![alt text](/assets/media/2026-07-02-virtual-enterprise-security-lab-p2/configure_network.png)
+ In this lab, the WAN interface is configured to obtain an IP address automatically using DHCP. It is responsible for providing Internet access, downloading software updates, installing packages, allowing internal clients to access external resources.
+ OPNsense is the core component of the lab. Every packet entering or leaving the enterprise network must pass through the firewall. Its responsibilites include routing traffic, firewall rule enforcement, IDS/IPS (Suricata), Proxy,... In a real enterprise, this device acts as the security gateway that protects the internal network from external threats.
+ LAN Interface is configured with the address 10.200.200.254/24. This ID address becomes the default gateway for every device inside the enterprise network. Every client sends its traffic to OPNsense first before reaching the Internet.
+ Virtual Layer 2 Switch connects all virtual machines within the internal network. It performs the same function as a physical Ethernet switch. Its responsibilities are connecting all devices within the LAN, forwarding Ethernet frames based on MAC addresses, creating a shared Layer 2 broadcast domain.
+ Virtual Servers : the left side of the topology contains several enterprise servers. 
+ Virtual Clients: the right side contains the client machines.

VMware Workstation supports four primary networking modes, each designed for a different use case:
+ LAN Segment: creates a completely isolated Layer 2 network for VMs.
+ Host-only: creates a private network that allows communication between the host and the virtual machines.
+ NAT (Network Address Translation) allows VMs to access the Internet through the Host by performing network address translation.
+ Bridged Networking: bridged mode connects a VM directly to the physical network.

Based on the analysis of the workflow and the available networking modes, we decided to use two distinct network adapters to simulate a realistic firewall environment.
1. Adapter 1 (WAN): Set to NAT mode.
2. Adapter 2 (LAN): Set to LAN Segment (named Internet). This configuration ensures that virtual machines can communicate only within the isolated LAN, while preventing direct network access to the host. As a result, if a virtual machine becomes infected with malware, the infection is confined to the isolated environment and is less likely to affect the host system through the network.


### Step 2: Installation via CLI
[Content in here](https://sysadmin.orbing.se/guides/install_and_configure_opnsense.html)
### Step 3: Assign Interfaces and IP Addresses
After rebooting, log in as root with password opnsense. You will be presented with a text-based menu.
#### 1. Assign Interfaces (Option 1)
Select Option 1. Based on the MAC addresses in VMware settings, map the interfaces:
+ WAN: `em0`.
+ LAN: `em1`.


#### 2. Set Interface IP Addresses (Option 2)
By default, LAN 1 might conflict with your home network IP range. We need to change it.


Select LAN interface.


Configure IPv4 address LAN interface via DNCP? n


IP Address: 10.200.200.254.


Subnet: /24.


Configure IPv6 address LAN interface via WAN tracking? n


Configure IPv6 address LAN interface via DHCP6? n


Do you want to enable the DHCP server on LAN? n


Do you want to change the web GUI protocol from HTTPS to HTTP? y


Restore web GUI access defaults? y


## How to configure VM Kali Linux
You can watch this [video](https://www.youtube.com/watch?v=bLNWyseSr_w&t=729s) to learn how to install Kali Linux on VMware.


I configured two network adapters (NAT and LAN Segment), but I'm having a problem. When I run `ip a`, neither `eth0` nor `eth1` has an IPv4 address.


I solved the issue with eth0 not having an IPv4 address by following this YouTube tutorial: [link](https://www.youtube.com/watch?v=PLLb-6lcMgI). For the remaining issue, I asked ChatGPT for help.


```
$ nmcli device status
DEVICE  TYPE      STATE                   CONNECTION         
eth0    ethernet  connected               Wired connection 1 
lo      loopback  connected (externally)  lo                 
eth1    ethernet  disconnected            --                 
$ sudo nmcli connection add type ethernet ifname eth1 con-name LAN
[sudo] password for kali: 
Connection 'LAN' (0a660842-b324-4b9c-bb67-4a03e0f37058) successfully added.
$ nmcli device status                                             
DEVICE  TYPE      STATE                                  CONNECTION         
eth0    ethernet  connected                              Wired connection 1 
eth1    ethernet  connecting (getting IP configuration)  LAN                
lo      loopback  connected (externally)                 lo    
$ sudo nmcli connection modify LAN \
ipv4.method manual \
ipv4.addresses 10.200.200.20/24 \
ipv4.never-default yes
$ sudo nmcli connection up LAN      
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/10)
```
> The ipv4.never-default yes option prevents NetworkManager from setting eth1 as the default gateway. This ensures that all Internet traffic continues to use eth0 (NAT), while eth1 is used only for communication within the local LAN Segment. As a result, it prevents routing conflicts on systems with multiple network interfaces.
{: .prompt-info }


Now, we can ping IP firewall:
```
$ ping 10.200.200.254
PING 10.200.200.254 (10.200.200.254) 56(84) bytes of data.
64 bytes from 10.200.200.254: icmp_seq=1 ttl=64 time=3.12 ms
64 bytes from 10.200.200.254: icmp_seq=2 ttl=64 time=0.587 ms
64 bytes from 10.200.200.254: icmp_seq=3 ttl=64 time=0.571 ms
^C
--- 10.200.200.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2031ms
rtt min/avg/max/mdev = 0.571/1.424/3.115/1.195 ms
```
## Installing the VMware Plugin
Login OPNsense Dashboard:
![alt text](/assets/media/2026-07-02-virtual-enterprise-security-lab-p2/login_opnsense.png)
System -> Firmware -> Status:
![alt text](/assets/media/2026-07-02-virtual-enterprise-security-lab-p2/status_system_firmware.png)
In plugin option, we choose os-vmware and install it!
![alt text](/assets/media//2026-07-02-virtual-enterprise-security-lab-p2/plugin-os-vmware.png)
Notification about installing sucessfully.
![alt text](/assets/media/2026-07-02-virtual-enterprise-security-lab-p2/install_plugin_sucessful.png)
## Reference
[1] https://sysadmin.orbing.se/guides/install_and_configure_opnsense.html


[2] https://sysadmin.orbing.se/guides/virtual_networking_vmware.html 