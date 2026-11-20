# Systemd Journal

## Introduction

Following our discussion on Linux logging mechanisms, let's delve deeper into one of the most powerful tools for modern systems: the systemd journal. This binary logging system addresses many limitations of traditional syslog, offering structured data that's easier to query and analyze during forensic investigations. By examining journal entries, we can uncover detailed timelines of system events, from boot sequences to service failures, helping to identify indicators of compromise like unauthorized process executions or configuration changes.

Systemd is a comprehensive suite of software tools that serves as the system and service manager for modern Linux distributions, acting as the init system (the first process started by the kernel with PID 1) to bootstrap the user space and manage subsequent processes. For initializing and managing the Linux system after the kernel boots, systemd offers features like:

- Service Management
- Resource Control
- Hardware and Device Management
- Timers and Scheduling
- Logging Integration
- System State Management

It replaces older init systems like SysVinit or Upstart and is the default in distributions such as Ubuntu, Fedora, Debian, and Arch Linux. It encompasses a wide range of components, including daemons for handling services, devices, mounts, timers, and logging, providing a unified framework for system initialization and management.

## Overview of Systemd Journal

The shortcomings of the aging syslog system have resulted in a number of security and availability enhancements. Many of these enhancements have been added to existing syslog daemons as non-standard features and never gained widespread use among Linux distributions. The systemd journal was developed from scratch as an alternative logging system with additional features missing from syslog, such as structured logging, forward-secure sealing, and efficient querying. Therefore, systemd-journald (or simply journald) is the logging component of systemd, functioning as a system service daemon that collects, processes, and stores logging data from the kernel, system services, applications, and other sources. It maintains structured, indexed journals in a binary format, which is more efficient than traditional text-based logs, and it conforms to standards like the Syslog protocol (RFC 5424) for message classification by priority and facility. The terms "journald" and "systemd-journald" are used interchangeably to describe this daemon.

### Key Components

To clarify its key components, here's a breakdown:

| Component | Description |
|---|---|
| Log originator | All of the messages produced by the kernel, initrd, services, etc. |
| Config files location | `/etc/systemd/journald.conf` |
| Daemon | systemd-journald (Service managed by systemd) |
| Local audit log | `/var/log/journal/*` `/run/log/journal/*` |
| Tool used to search | `journalctl` |

One of the benefits of using a binary journal for logging is the ability to view log records in local time and in UTC if you want. By default, systemd will display results in local time. Lab system is configured to use UTC timezone.

```shellsession
linuxforensics@ubuntu:~$ sudo timedatectl site-timezone UTC
```

You can run the journalctl utility to show initial results (when used alone, every journal entry that is in the system will be displayed within a pager (usually less) for you to browse. The oldest entries will be up top):

```shellsession
linuxforensics@ubuntu:~$ journalctl
```
<img width="654" height="753" alt="image" src="https://github.com/user-attachments/assets/87405235-46db-405c-8f97-4d43db940916" />

## Journalctl

Journalctl is a command-line utility that is using systemd that allows us to query, view, and manage logs collected by the systemd-journald daemon, providing a centralized interface to access structured binary journal files containing detailed system events, kernel messages, service outputs, and metadata like timestamps, priorities, and process IDs. Let’s take a look at some useful journalctl commands:

### Journalctl Commands

| Command | Description |
|---|---|
| `journalctl --utc` | Display timestamps in UTC |
| `journalctl -b` | Display logs from current boot |
| `journalctl --list-boots` | List previous boots |
| `journalctl -b -1` | See journal from the previous boot (you can use boot ID instead -1) |
| `journalctl --utc -D /home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/journal/894062f9af204645a289e8016977fe6c` | Use external journal folder to retrieve results from |
| `journalctl --utc --since "2023-10-15 18:00:00" --until "2023-10-15 19:00:00" -D /home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/journal/894062f9af204645a289e8016977fe6c` | Use time windows for your logs (format: YYYY-MM-DD HH:MM:SS) |
| `journalctl --utc --since "2023-10-15 18:00:00" --until "2023-10-15 19:00:00" -D /home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/journal/894062f9af204645a289e8016977fe6c/ -u httpd.service` | Filter results by unit (http.service as an example) |
| `journalctl --utc --since "2023-10-15 18:00:00" --until "2023-10-15 19:00:00" -D /home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/journal/894062f9af204645a289e8016977fe6c/ -u httpd.service _PID=27804` | Filter results by Process ID (PID) |
| `journalctl /usr/bin/bash` | Filter results by executable |
| `journalctl -p err --no-pager` | Filter results by priority (You can use either the priority name or its corresponding numeric value) • 0: emerg • 1: alert • 2: crit • 3: err • 4: warning • 5: notice • 6: info • 7: debug |
| `journalctl -o json-pretty` | Output journalctl in json or any other formats: • cat: Displays only the message field itself. • export: A binary format suitable for transferring or backing up. • json: Standard JSON with one entry per line. • json-pretty: JSON formatted for better human-readability • json-sse: JSON formatted output wrapped to make add server-sent event compatible • short: The default syslog style output • short-iso: The default format augmented to show ISO 8601 wallclock timestamps. • short-monotonic: The default format with monotonic timestamps. • short-precise: The default format with microsecond precision • verbose: Shows every journal field available for the entry, including those usually hidden internally. |

### Trusted Journal Fields

You can use `man systemd.journal-fields` to identify fields that can be used for search:

| Trusted Journal Fields | Description |
|---|---|
| `_PID`, `_UID`, `_GID` | The PID, UID, and group ID of the process the journal entry originates from, formatted as a decimal string. Note that entries obtained via "stdout" or "stderr" of forked processes will contain credentials valid for a parent process (that initiated the connection to systemd-journal). |
| `_COMM`, `_EXE`, `_CMDLINE` | The name, executable path, and command line of the process the journal entry originates from. |
| `_CAP_EFFECTIVE=` | The effective capabilities(7) of the process the journal entry originates from. |
| `_AUDIT_SESSION`, `_AUDIT_LOGINUID` | The session and login UID of the process the journal entry originates from, as maintained by the kernel audit subsystem. |
| `_SYSTEMD_CGROUP=`, `_SYSTEMD_SLICE=`, `_SYSTEMD_UNIT=`, `_SYSTEMD_USER_UNIT=` | The control group path in session, systemd hierarchy owner UID, the systemd slice unit name, the unit name in the systemd user manager (if any), the owner ID (if any) of the systemd user unit or systemd session (if any), and the owner UID of the systemd user. |
| `_SELINUX_CONTEXT=` | The SELinux security context (label) of the process the journal entry originates from. |

### Usage Example

```shellsession
0xwillback@htb[/htb]$ journalctl --utc --since "2023-10-15 18:00:00" --until "2023-10-15 19:00:00" -D /home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/journal/894062f9af204645a289e8016977fe6c/ -u httpd.service _PID=27804 -o json-pretty
```

<img width="646" height="593" alt="image" src="https://github.com/user-attachments/assets/cebf4ccd-ec29-494a-877d-c0433664f0a2" />

To enhance forensic analysis, consider exporting journals for offline review. Use `journalctl -o export > journal.export` to create a binary export file, which can be imported elsewhere with `journalctl --import journal.export`. For persistence, configure `/etc/systemd/journald.conf` to set `Storage=persistent`, ensuring logs survive reboots in `/var/log/journal/`. In investigations, combine filters for targeted searches, like `journalctl -u ssh.service -p err` to spot SSH errors. Tools like `journalctl --vacuum-time=2weeks` can clean old entries, but in forensics, disable this to preserve data. For remote journals, enable forwarding with SystemLogSocket in journald.conf, sending to a central server for aggregation.

## 🔑 Key Takeaway

The systemd journal provides structured, indexed binary logs that support detailed forensic investigations. Using `journalctl`, investigators can filter entries by boot, time range, service, process ID, executable, and priority. Preserving journal data, exporting it for offline analysis, and reviewing relevant metadata can help reconstruct system activity and identify suspicious events.
