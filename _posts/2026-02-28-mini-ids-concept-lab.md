---
title: "Mini IDS Concept Lab: Building an Anomaly-Based Detection System"
date: 2026-02-28 09:00:00 +0300
categories: [Cybersecurity, Lab Tutorial]
tags: [ids, wireshark, network-security, kali-linux, virtualbox]
image:
  path: /assets/img/ids-concept-lab/12-ids-avatar.jpg
description: A hands-on guide to building and understanding anomaly-based intrusion detection systems using VirtualBox, Kali Linux, and Wireshark.
---

## Introduction

Intrusion Detection Systems (IDS) play a critical role in modern network security by monitoring traffic for suspicious activity and potential attacks. Organizations rely on IDS solutions to detect anomalies, malicious patterns, and unauthorized access attempts in real time.

There are two primary detection approaches: **signature-based detection**, which matches known attack patterns, and **anomaly-based detection**, which identifies deviations from normal traffic behavior.

This lab focuses on the anomaly-based detection concept. The objective was to establish a baseline of normal network traffic and then simulate abnormal activity to observe how traffic patterns change. Using Wireshark within a controlled virtual environment, traffic behavior was analyzed similarly to how enterprise-level IDS solutions such as Snort operate.

### Lab Objectives

- Configure a controlled virtual network environment using VirtualBox
- Capture and analyze normal network traffic
- Establish a traffic baseline
- Simulate suspicious network behavior
- Analyze traffic anomalies using Wireshark
- Design a simple detection rule based on observed deviations

## Lab Environment

This lab was conducted using:

- **VirtualBox** hypervisor
- **Kali Linux** (Attacker / Analyzer machine)
- **Windows 10** (Victim machine)
- **Wireshark** (Packet analysis tool)

Both virtual machines were configured within an isolated Internal Network to ensure no interference with the host system or external networks.

![Windows 10 VM Configured in VirtualBox](/assets/img/ids-concept-lab/01-windows10-vm.png)
*Figure 1: Windows 10 VM Configured in VirtualBox*

![Kali VM Configured in VirtualBox](/assets/img/ids-concept-lab/02-kali-vm.png)
*Figure 2: Kali VM Configured in VirtualBox*

## Network Configuration

### Step 1: Configure Internal Network

Inside VirtualBox:

1. Selected Kali VM, went to the Settings option then the Network tab
2. Under Adapter 1, attached to the **Internal Network**
3. Name set as **IDS_LAB**

The same process was repeated for the Windows VM.

![Internal Network Configuration for Kali Linux](/assets/img/ids-concept-lab/03-kali-network.png)
*Figure 3: Internal Network Configuration for Kali Linux*

![Internal Network Configuration for Windows 10](/assets/img/ids-concept-lab/04-windows-network.png)
*Figure 4: Internal Network Configuration for Windows 10*

### Step 2: Assign IP Addresses

After starting both machines, I checked the IP configuration on Kali Linux using the `ip a` command. No IPv4 address was assigned by default, so I manually assigned one.

![Manually assigning IP address for Kali](/assets/img/ids-concept-lab/05-kali-ip.png)
*Figure 5: Manually assigning IP address for Kali*

On Windows 10, I set the manual IP from the Control Panel.

![Manually assigning IP address for Windows 10](/assets/img/ids-concept-lab/06-windows-ip.png)
*Figure 6: Manually assigning IP address for Windows 10*

### Testing Connectivity

From Kali, I pinged the Windows 10 IP address to test connectivity.

![Successful ping of Windows 10 from Kali](/assets/img/ids-concept-lab/07-ping-test.png)
*Figure 7: Successful ping of Windows 10 from Kali*

## Baseline Traffic Capture

### Step 1: Starting Wireshark on Kali

![Initial packet capture](/assets/img/ids-concept-lab/08-baseline-capture.png)
*Figure 8: Initial packet capture*

No packets were actively being captured on eth0 because Windows 10 had not yet sent any traffic.

### Step 2: Generating Normal Traffic

On Windows, I pinged the Kali IP address and then opened a web browser. Traffic was captured for approximately 5 minutes in Wireshark.

### Step 3: Analyzing Baseline

![Baseline Protocol Distribution](/assets/img/ids-concept-lab/09-baseline-protocol.png)
*Figure 9: Baseline Protocol Distribution*

According to the statistics of the capture:

- **ICMP**: 50.0%
- **ARP**: 20.0%
- **TCP**: 0.0%

The absence of TCP traffic was expected in this clean baseline because:
- No web browsing occurred during baseline capture
- No file transfers or connection-oriented services were active
- Windows 10 was idle except for initial ping tests

**Average packets per second** was approximately **0.05 packets/second**:
- Total Packets: 20
- Capture Duration: 386 seconds
- Average PPS = 20 ÷ 386 = 0.0518 pkts/s ≈ 0.05 pkts/s

### Baseline Traffic Analysis

During normal operation, traffic remained consistent and low in volume. ICMP activity was minimal and occurred only during manual ping testing. ARP traffic appeared periodically for address resolution. No sudden spikes or abnormal bursts were observed.

This established a predictable baseline for comparison.

## Suspicious Traffic Simulation

### Step 1: Restarting Capture in Wireshark

I restarted the capture process in Wireshark.

### Step 2: Simulating ICMP Flood

The following command was executed to simulate an ICMP flood:

```bash
sudo ping -i 0.002 192.168.100.20
```

**Command breakdown:**
- `sudo`: Runs with root privileges (required for flood timing)
- `ping`: The ping utility
- `-i 0.002`: Sets the interval between packets to 0.002 seconds (500 packets per second)
- `192.168.100.20`: Target Windows 10 machine

This creates a high-speed flood of ICMP echo requests, overwhelming the target with ~500 packets per second.

### Step 3: Observing in Wireshark

A display filter for ICMP was applied in Wireshark to isolate only ping-related traffic. The I/O Graph tool was then used to visualize the rate of ICMP packets over time.

![Traffic spike during ICMP Flood](/assets/img/ids-concept-lab/11-io-graph.png)
*Figure 11: Traffic spike during ICMP Flood*

The I/O Graph revealed:

- **Peak Rate**: ~1000 packets/second (reached at ~5 seconds)
- **Duration of Spike**: Approximately 7.5 seconds (from 2.5s to 10s)
- **Total Packets in Spike**: ~7,500 packets (estimated)
- **Comparison to Baseline**: Baseline average was 0.05 pkts/s; spike reached 1000 pkts/s which was a **20,000% increase**

This dramatic deviation confirms an anomalous event that any IDS would flag.

## Traffic Analysis and IDS Logic

During the ICMP flood simulation, a dramatic increase in ICMP echo requests was observed. Compared to the baseline capture, packet rates increased significantly within a short time frame.

This behavior represents an **anomaly** - a deviation from normal operational traffic.

Based on the observations, the following simple detection rule was designed:

> **If ICMP packet rate exceeds 50 packets per second, flag as suspicious activity.**

This rule demonstrates the core principle of anomaly-based intrusion detection systems.

### Threshold Determination

- **Baseline ICMP rate**: 0.05 pkts/s
- **Attack ICMP rate**: 1000 pkts/s
- **Selected threshold**: 50 pkts/s

Setting the threshold at 50 pkts/s provides:

- **1000x** baseline rate (ensures no false positives)
- **20x** below attack rate (catches attack early)
- **95%** safety margin below peak attack rate

This threshold balances **sensitivity** (catches attacks) with **specificity** (avoids false alarms).

## Lab Limitations

- **Simplified Environment**: Real networks have hundreds of devices and mixed traffic
- **Single Attack Type**: Only ICMP flood was tested; real IDS must detect many attack vectors
- **No Encryption**: All traffic was plaintext; real networks often have encrypted traffic
- **Manual Analysis**: Enterprise IDS uses automated alerts, not manual Wireshark inspection
- **No Response Mechanism**: This lab only detects; real IDS would also alert or block

## Findings

The lab demonstrated that abnormal traffic patterns are visually and statistically distinguishable from normal behavior. Establishing a baseline was essential in identifying the anomaly. Without baseline data, detection of suspicious behavior would be significantly more difficult.

## Conclusion

This lab successfully demonstrated the principles of anomaly-based intrusion detection within a controlled virtual environment. By establishing a baseline of normal traffic and introducing simulated abnormal behavior, clear deviations were identified using Wireshark analysis tools.

The exercise highlighted the importance of traffic monitoring, baseline profiling, and threshold-based detection rules in cybersecurity operations. Although simplified, this simulation reflects how enterprise IDS systems detect suspicious behavior and alert security teams.

This lab strengthened practical understanding of traffic analysis, anomaly recognition, and IDS design concepts.

