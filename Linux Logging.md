# Linux Logging

## Introduction

Logging mechanisms are crucial in uncovering the sequence of events during an incident. Logs capture everything from system boot processes to user authentications and application errors, providing a chronological record that helps investigators piece together what occurred. By analyzing these logs, we can identify anomalies, trace malicious activities, and correlate events across the system. Similar to how we examined persistence mechanisms in the previous section, understanding logging is essential for detecting hidden threats and building a comprehensive incident timeline.

## Common Linux Logging Mechanisms

There are several notable mechanisms of logging in Linux:

- **Syslog:** Syslog is a standard logging protocol used to collect and send log messages within a network. Linux systems often use syslog to store log data in various log files, typically located in the `/var/log/` directory. Common log files include `/var/log/messages`, `/var/log/auth.log`, and `/var/log/syslog`. These logs can provide valuable information about system and application activities, such as failed login attempts or service startups.

- **Systemd Journal:** Systemd, a common init system on many Linux distributions, uses the systemd journal for logging. Journalctl is the command-line tool for querying and examining these logs. The journal provides structured and more comprehensive information compared to traditional flat text log files, including metadata like timestamps, process IDs, and severity levels.

- **Auditd:** The Linux Audit framework allows for detailed monitoring of system activities, including file access, process creation, and user authentication. Auditd is the user-space component, and it logs its data to `/var/log/audit/audit.log`. This is a powerful tool for tracking changes and potential security incidents, often used in compliance-heavy environments.

- **SysmonForLinux:** This is a tool that monitors and logs system activity, including processes, network connections and file system writes similar to what Sysmon for Windows does. SysmonForLinux logs events into `/var/log/syslog` using XML format. It's particularly useful for endpoint detection and response (EDR) scenarios.

While Syslog and Systemd Journal are typically built in, Auditd may require enabling or additional configuration on minimal installs, and SysmonForLinux needs separate software installation from repositories like those provided by Microsoft.

## Syslog

The traditional logging system on Unix and Unix-like operating systems, such as Linux, is syslog. Syslog enables the separation of software that generates log messages (like applications, services, or the kernel) from the systems that store, process, or forward those messages. In Linux, Syslog is implemented through daemons (background processes) like syslogd, rsyslog, or syslog-ng, which handle the collection and management of logs. It's essentially a way for the system to record events, errors, and other important information in a structured manner. While traditional Syslog refers to the protocol defined in RFC 5424, in Linux contexts, it often encompasses the entire logging ecosystem, including the daemon and configuration files. Note that modern Linux distributions (those that are using systemd) may integrate or replace parts of Syslog with tools like the Systemd Journal (systemd-journald), but Syslog remains widely used for compatibility and flexibility.

Syslog operates by having applications, services, or the kernel generate log messages that are sent to a daemon like rsyslogd or syslog-ng via local sockets (e.g., `/dev/log`), the logger command, or network protocols (UDP/TCP on port 514 for remote logging). These messages follow a standardized format including a priority code (combining facility, such as kern or auth, with severity levels from emerg to debug), a header with timestamp and hostname, and the actual content. The daemon processes incoming messages according to rules defined in configuration files (e.g., `/etc/rsyslog.conf`), where selectors match facility.priority patterns to actions like writing to specific log files in `/var/log/` (such as `/var/log/syslog` for general logs or `/var/log/auth.log` for authentication events), forwarding to another server for centralized logging, discarding, or piping to external programs. Logs are stored persistently, with tools like logrotate handling rotation and compression to manage disk space, ensuring efficient, categorized recording of system events for troubleshooting, monitoring, and auditing while supporting both local and networked environments.

Syslog is typically implemented as a daemon (also known as a collector) that listens for log messages from multiple sources, such as packets arriving over network sockets (UDP port 514), local named pipes, or syslog library calls.

### Key Components

To make this clearer, here's a breakdown of key components:

| Component | Description |
|---|---|
| Log originator | Programs with syslog support kernel messages |
| Config files location | `/etc/rsyslogd.conf` `/etc/rsyslogd.d/conf` `/etc/syslog-ng/` |
| Daemon | `/usr/sbin/rsyslogd` (Service started by systemd) |
| Network log host | UDP port 514 (Configured with @host) |
| Local logfiles | `/var/log/*` (By facility and severity) |

### Viewing Recent Syslog Entries

For example, to view recent syslog entries on a Debian-based system:

```shellsession
@htb[/htb]$ tail -n 10 /var/log/syslog
```


<img width="762" height="395" alt="image" src="https://github.com/user-attachments/assets/140fbba5-762e-4441-9682-1ee0ba191a32" />

This output can reveal patterns like unauthorized access or cron job executions. In forensics, we often use tools like grep to filter for specific events, e.g., `grep "Failed password" /var/log/auth.log` to spot brute-force attempts.

## Extending to Additional Mechanisms

There are also some advanced or specialized logging options commonly encountered in Linux forensics:

- **Kernel Ring Buffer (dmesg):** This logs kernel messages, such as hardware detections or driver issues, stored in a circular buffer. Access it via the `dmesg` command or `/var/log/dmesg` on boot. Useful for investigating boot-time anomalies or device attachments.

- **Application-Specific Logs:** Many services maintain their own logs, like Apache (`/var/log/apache2/access.log`) or MySQL (`/var/log/mysql/error.log`). These provide granular details on web traffic or database queries, often rotated via logrotate in `/etc/logrotate.d/`.

- **ELK Stack or Splunk Integration:** In enterprise setups, logs are forwarded to centralized systems like Elasticsearch, Logstash, Kibana (ELK), or Splunk for aggregation and analysis. Check `/etc/rsyslog.d/` for forwarding configs to detect exfiltration or monitoring setups.

### Checking Kernel Logs

For instance, to check kernel logs, we can use the following command:

```shellsession
@htb[/htb]$ dmesg | tail -n 5
```

<img width="714" height="194" alt="image" src="https://github.com/user-attachments/assets/2c853adb-47e5-43d0-ae83-f51ea00fcb9f" />

When investigating logs, always preserve originals by copying to a forensic image. Use tools like log2timeline from Plaso to create super timelines correlating logs with file timestamps. Rotating and compressing logs (via `/etc/logrotate.conf`) can hide older events, so check archived files in `/var/log/*.gz`.

## 🔑 Key Takeaway

Linux logging mechanisms provide essential evidence for reconstructing system activity and investigating security incidents. Syslog, Systemd Journal, Auditd, and SysmonForLinux record different types of events, while kernel and application-specific logs provide additional context. Preserving original logs, examining rotated archives, and correlating timestamps across sources help investigators build a comprehensive incident timeline.
