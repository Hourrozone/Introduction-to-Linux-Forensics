# SysmonForLinux in Linux Forensics

## Introduction

SysmonForLinux is an open-source system monitoring tool developed by Microsoft as part of the Sysinternals suite, bringing the powerful event tracing capabilities originally designed for Windows to Linux environments. It serves as a robust solution for capturing and logging detailed system activities, enabling security professionals, system administrators, and incident responders to gain deep visibility into potential threats and operational behaviors on Linux systems. By leveraging extended Berkeley Packet Filter (eBPF) technology, SysmonForLinux efficiently monitors key events without significant performance overhead, making it adaptable for both standalone servers and large-scale deployments in cloud or enterprise settings.

At its core, this tool helps in detecting advanced persistent threats (APTs), such as fileless malware that operates in memory without leaving traditional footprints, or lateral movement techniques where attackers pivot between systems to escalate privileges or exfiltrate data. It achieves this by logging a wide array of events, including process creations and terminations (tracking command lines, parent-child relationships, and hashes), network connections (capturing source/destination IPs, ports, and protocols), file system modifications (like creations, deletions, or attribute changes), and more. These logs are context-rich, often including metadata like timestamps, user IDs, and integrity levels, which provide invaluable insights during forensic investigations or real-time threat hunting.

In terms of operation, it's installed as a service and device driver that persists across reboots, ensuring continuous monitoring. Logs are primarily directed to `/var/log/syslog` by default, but can be configured to integrate with other logging systems like systemd-journald or forwarded to centralized SIEM platforms such as Azure Sentinel or Elastic Security for aggregation and analysis. Configuration is handled through XML files, where users define rules for event filtering, such as including or excluding specific processes, paths, or behaviors, to focus on high-value activities while minimizing noise. For viewing logs, the included SysmonLogView utility parses and formats the output for better readability, allowing queries based on event types or keywords.

What makes SysmonForLinux a game-changer is its ability to complement native Linux tools like Auditd or Syslog by providing more granular, security-focused telemetry that traditional logging might miss. For example, in incident analysis, its logs can reveal anomalies such as unexpected process injections, suspicious registry-like changes in configuration files, or outbound connections to command-and-control (C2) servers, helping teams reconstruct attack chains and respond swiftly. Installation is straightforward via Microsoft's Linux repositories for distributions like Ubuntu, Red Hat, or Debian, with build options available for custom setups. As of 2025, ongoing updates have enhanced its compatibility with modern kernels and eBPF advancements, ensuring it remains a vital asset in hybrid Windows-Linux environments for proactive threat detection and compliance auditing. Whether you're bolstering endpoint detection and response (EDR) capabilities or simply seeking better system observability, SysmonforLinux stands out as a versatile, no-cost addition to your security toolkit.

## Overview of SysmonForLinux

It uses eBPF programs to hook into kernel events, automatically discovering kernel offsets or relying on BPF Type Format (BTF) for precise tracing on supported systems. From an event-tracing perspective, eBPF allows us to write event-driven programs and have pre-defined hooks into operations such as system calls, network connections, and file write/read. You can find out how to automate the deployment of SysmonForLinux here.

<img width="3125" height="2396" alt="image" src="https://github.com/user-attachments/assets/3ad41e9b-157a-467d-8203-02de5ba125b9" />

### Key Components

To clarify its key components, here's a breakdown:

| Component | Description |
|---|---|
| Underlying Technology | eBPF for event-driven kernel hooks |
| Log Location | `/var/log/syslog` (XML format) |
| Configuration | XML files (e.g., `sysmonconfig.xml`) for filtering events |
| Daemon | Sysmon (Installed as a service) |
| Parsing Tool | SysmonLogView for converting XML to human-readable format |
| Installation | Via Microsoft repositories or build from source on GitHub |

## Supported Events

Currently, SysmonForLinux supports logging of the following events:

| Event ID | Description |
|---|---|
| 1 | Logs when a new process is created. |
| 3 | Logs TCP/UDP connections on the machine. |
| 4 | Logs the state of the Sysmon service (started or stopped). |
| 5 | Logs when a process terminates. |
| 9 | Logs when a process conducts reading operations, from the drive. |
| 11 | Logs when a file is created or overwritten. |
| 16 | Logs when the local Sysmon configuration is updated. |
| 23 | Logs when a file is deleted by a process. |

Note that this is a subset of Windows Sysmon events; future updates may expand support.

## Installation and Configuration

To install SysmonForLinux, add Microsoft's repository for your Ubuntu distro:

```shellsession
cry0l1t3@ubuntu:~$ wget -qO- <https://packages.microsoft.com/keys/microsoft.asc> | sudo apt-key add - 
cry0l1t3@ubuntu:~$ sudo add-apt-repository "deb [arch=amd64] <https://packages.microsoft.com/ubuntu/$>(lsb_release -rs)/prod $(lsb_release -cs) main"
cry0l1t3@ubuntu:~$ sudo apt update && sudo apt install sysmonforlinux -y
cry0l1t3@ubuntu:~$ sudo sysmon -i sysmonconfig.xml   # <- Start with this command for a custom config, or without for defaults
```

Configuration uses XML files to filter events, e.g.:

```xml
<Sysmon schemaversion="4.90">
  <EventFiltering>
    <ProcessCreate onmatch="exclude">
      <Image>/bin/ls</Image>
    </ProcessCreate>
  </EventFiltering>
</Sysmon>
```

We can update configs with:

```shellsession
linuxforensics@ubuntu:~$ sudo sysmon -c newconfig.xml
```

For forensics, use configs from repos like SwiftOnSecurity's (adapted for Linux) to focus on high-value events.

## SysmonLogView

SysmonLogView is designed to enhance the usability of Sysmon's logging output on Linux systems. It primarily serves to convert the XML-formatted event logs generated by Sysmon (`/var/log/syslog`) into a more human-readable, structured text format, making it easier for us to parse, analyze, and troubleshoot system activities without dealing with raw XML clutter. It works by reading syslog data from standard input (stdin) and writing processed, filtered output to standard output (stdout), allowing easy integration into pipelines or scripts for real-time monitoring and/or batch processing.

<img width="613" height="667" alt="image" src="https://github.com/user-attachments/assets/97af5c41-de50-4093-894d-a6dff4aca305" />

<img width="631" height="413" alt="image" src="https://github.com/user-attachments/assets/4aca1bc9-26d3-401c-82e7-7a3606da36b0" />

## Forensic Tips

In investigations, pipe logs to tools like jq for JSON-like parsing: `grep "sysmon" /var/log/syslog | jq ..` Forward to a SIEM (e.g., ELK) by configuring rsyslog to send Sysmon events. Rotate logs via `/etc/logrotate.d/syslog` to preserve history. However, you need to ensure eBPF supports kernel 4.18+. You can test configs with `sysmon -? config`.

For example, filter for process creations:

```shellsession
linuxforensics@ubuntu:~$ sudo tail -f /var/log/syslog | grep "EventID>1" | sudo /opt/sysmon/sysmonLogView -f EventID=1
```

<img width="683" height="549" alt="image" src="https://github.com/user-attachments/assets/14abb687-da43-48dd-944d-315f53ce2fb6" />

You can also pair SysmonForLinux with Auditd for more comprehensive coverage. SysmonForLinux excels in process/network events, while Auditd handles syscalls. Additionally, with SysmonLogView you can export the logs to CSV for analysis: `sysmonLogView -csv output.csv < syslog`.

## 🔑 Key Takeaway

SysmonForLinux uses eBPF to capture detailed Linux system telemetry, including process activity, network connections, and file operations. Its XML-based configuration allows filtering of events, while SysmonLogView makes the resulting logs easier to analyze. Combining SysmonForLinux with Auditd and centralized logging provides broader visibility for threat detection and forensic investigations.
