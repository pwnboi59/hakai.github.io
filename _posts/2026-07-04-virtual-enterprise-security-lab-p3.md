---
title: "Virtual Enterprise Security Lab - Part 3: Suricata IDS/IPS Installation on Opnsense"
date: 2026-07-04
categories: [Cybersecurity, Home Lab]
---

## Configuring Suricata IDS/IPS in OPNsense 
### 1. Disable Hardware Offloading (Critical First Step)
Suricata needs to inspect raw packets before they are processed.
If your network interface card (NIC) is using hardware offloading, Suricata will not be able to see the complete packet data. As a result, the IDS/IPS may miss threats or fail to analyze network traffic accurately. 
![alt text](/assets/media/2026-07-04-virtual-enterprise-security-lab-p3/workflow.png)
Looking at the packet processing workflow, our goal is to ensure that packets remain as intact and visible as possible before they are transmitted by the NIC. Hardware offloading features should be disabled so that packet processing is performed by the operating system instead of the network interface card. This allows the firewall and Suricata to inspect the complete packet contents, apply firewall rules, perform stateful inspection, and detect malicious traffic accurately before the packets leave the system. Although this increases CPU utilization, it significantly improves the effectiveness and reliability of IDS/IPS detection. So we have to disable Hardware CRC, Hardware TSO, Hardware LRO.

## Reference
[1] https://docs.opnsense.org/manual/interfaces_settings.html
 

[2] https://corelab.tech/opnsense-ids-ips-suricata-inline-vs-divert-mode/