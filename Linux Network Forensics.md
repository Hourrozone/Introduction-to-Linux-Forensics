# Linux Network Forensics

## Scenario Overview

Let’s assume your cybersecurity team is investigating an incident at a medium-sized e-commerce company that operates an online store hosted on a Linux-based web server. The server runs Ubuntu 22.04 LTS, configured with Apache 2.4 as the web server, PHP 8.1 for backend scripting, and MySQL for the database. It's a virtual machine hosted on AWS EC2, accessible via SSH with key-based authentication for the root and a limited webadmin user. The server handles moderate traffic, serving product pages, user logins, and payment processing integrations. Firewall rules (using ufw) allow inbound traffic on TCP ports:

- **80 (HTTP)**
- **443 (HTTPS)**
- **22 (SSH from specific IPs).**

Auditd has been pre-installed and configured to log system calls, file accesses, and execve events for enhanced monitoring as part of the company's proactive security measures following a previous minor incident.

The server is not heavily segmented. It runs everything under a single instance to keep costs down, making it a prime target for lateral movement if compromised. No intrusion detection systems like Snort or Fail2Ban are active, but systemd journaling is enabled for service logs. The company uses Velociraptor for endpoint forensics, deployed via an offline collector for rapid artifact gathering without live agent installation.

## Case Background

The company's network monitoring tool (a basic Splunk setup) flagged unusual outbound traffic from the web server to known cryptocurrency mining pools. The spikes occurred between 6:00 PM and 7:00 PM host time (UTC-4, as the server is set to Eastern Time). CPU usage surged to 90% during this window, causing site slowdowns and customer complaints. Initial checks showed no obvious cron jobs or persistent services, but a quick netstat revealed ephemeral connections that terminated shortly after. Fearing a breach, the security lead initiated a forensic collection:

- Memory dump using LiME (Linux Memory Extractor)
- Apache access/error logs
- Velociraptor offline collector run via SSH.

The collector gathered artifacts including, `/var/log/audit/audit.log`, systemd journal (parsed to JSON), raw uploads, and specific user files like root's `.viminfo`.

Suspecting a web application vulnerability, the security lead hands you the case artifacts, stored on your forensic workstation (your target) at the paths provided. Your goal is to reconstruct the attack chain, identify the entry point, extract any fileless malware remnants from memory, and build a timeline of the actor's actions. This could involve a supply-chain attack via a compromised dependency or a direct exploit of the Apache server.

### Collected Artifacts

The collected artifacts from the system can be found at the following locations on the target:

| Artifact | Location |
|---|---|
| Scenario case artifacts general folder | `/home/linuxforensics/Desktop/cases/scenario1` |
| Memory dump | `/home/linuxforensics/Desktop/cases/scenario1/web-server-dump` |
| Apache web application logs | `/home/linuxforensics/Desktop/cases/scenario1/apache` |
| Velociraptor offline collection general folder | `/home/linuxforensics/Desktop/cases/scenario1/collection` |
| Velociraptor offline collection parsed results | `/home/linuxforensics/Desktop/cases/scenario1/collection/results` |
| Velociraptor collected raw artifacts | `/home/linuxforensics/Desktop/cases/scenario1/collection/uploads` |
| audit.log | `/home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/audit/audit.log.1`, `/home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/audit/audit.log` |
| Systemd parsed results (journalctl) | `/home/linuxforensics/Desktop/cases/scenario1/journal.json` |
| Root user .viminfo file | `/home/linuxforensics/Desktop/cases/scenario1/root_viminfo` |

## Host Information

<img width="651" height="281" alt="image" src="https://github.com/user-attachments/assets/a69636a2-c983-4779-b3fa-23307d72edc1" />

### Timezone of the Compromised System

Timezone of the compromised system is ETC/UTC:

<img width="495" height="110" alt="image" src="https://github.com/user-attachments/assets/352276aa-a544-423c-82b9-1cc1a9bd41e1" />

## Network

Our investigation starts with the analysis of network artifacts, and we will begin by utilizing Volatility to examine TCP connections from the memory dump.

<img width="684" height="716" alt="image" src="https://github.com/user-attachments/assets/a32a652f-ed23-436e-b0c5-9da18235356c" />

We can find multiple suspicious network connections:

<img width="670" height="278" alt="image" src="https://github.com/user-attachments/assets/58b48d0e-951b-472b-895c-346105ff7176" />

The first one and the most suspicious is 51.75.64.249 IP address. Quick check on VirusTotal showed no significant results:

https://www.virustotal.com/gui/ip-address/51.75.64.249/detection

<img width="1600" height="658" alt="image" src="https://github.com/user-attachments/assets/bc166ade-6fb3-41aa-b1c9-0aa1fa7175e8" />

As an alternative for examining network artifacts, we can also review the Linux.Network.NetstatEnriched artifact obtained by the Velociraptor collector.

<img width="700" height="830" alt="image" src="https://github.com/user-attachments/assets/87e6d9a0-3154-4df3-b596-4228be187b88" />

From there, we can determine that the process connecting to this IP was executed via a cron job. From that artifact (use without grep), we can also ascertain that Apache is running on that system:

<img width="723" height="865" alt="image" src="https://github.com/user-attachments/assets/1dc1700f-4c7a-4ba6-bdfc-45bebb4a5d79" />

Let’s review in the Velociraptor collection the other four TCP connections obtained from the memory dump with a destination port of TCP/21 and a remote host IP of 192.168.152.180 (in the Velociraptor collection, we can find only one active connection):

<img width="733" height="829" alt="image" src="https://github.com/user-attachments/assets/a56854bb-7154-48c5-bdec-687ce62b7d81" />

We can observe the execution of the shell command (Pid:28645) that is frequently employed by malicious actors to establish a reverse shell.

## 🔑 Key Takeaway

Linux network forensics combines memory analysis with endpoint artifacts to identify suspicious connections and trace the processes responsible for them. In this scenario, Volatility reveals suspicious network connections, while Velociraptor's `Linux.Network.NetstatEnriched` artifact helps associate a connection with a cron job and identify Apache activity. Reviewing additional TCP connections and process execution evidence can help reconstruct the attack chain and investigate potential reverse-shell activity.
