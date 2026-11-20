# Auditd

## Introduction

Auditd is a powerful auditing framework that goes beyond standard logging by capturing granular, security-relevant events at the kernel level, making it invaluable for detecting intrusions, compliance violations, or unauthorized changes. In incident response, Auditd logs can reveal file tampering, syscall abuses, or user modifications that might evade other mechanisms, allowing us to reconstruct attack paths with precision. It operates by starting as a service (e.g., via `systemctl start auditd`) and communicating with the kernel's audit subsystem through a netlink socket to receive event messages. It applies predefined audit rules to filter and log relevant events, writing them in a structured format to `/var/log/audit/audit.log` (or other configured locations), with features like log rotation and buffering to handle high volumes. Configuration is managed via `/etc/audit/auditd.conf`, and it can be controlled with tools like `auditctl` for runtime adjustments. If the log fills up, it can trigger actions like halting the system for security.

## Overview of Auditd

The Linux Audit subsystem provides a secure logging framework that is used to capture and record security relevant events. Linux auditing is a kernel feature that generates an audit trail based on a set of rules. It has similarities to other logging mechanisms, but it is more flexible, granular, and able to log file access and system calls. The auditctl program loads rules into the kernel, and the auditd daemon writes the audit records to disk.

### Key Components

To clarify its key components, here's a breakdown:

| Component | Description |
|---|---|
| Audit rules | `/etc/audit/audit.rules` `/etc/audit/rules.d/*.rules` |
| Config files location | `/etc/audit/auditd.conf` |
| Daemon | `/usr/sbin/auditd` |
| Local audit log | `/var/log/audit/audit.log` |
| Tool used to configure rules | `auditctl`, `augenrules` reads in the `/etc/audit/rules.d/` and compiles them into an `audit.rules` file. |
| Tool used to search | `aureport` and `ausearch` reads `audit.log` file |
| Recommended basic configuration file | https://github.com/Neo23x0/auditd/blob/master/audit.rules |

To get started with Auditd, first ensure it's installed (e.g., `sudo apt install auditd` on Debian-based systems or `sudo yum install audit` on RHEL-based systems). Then, enable the service with `sudo systemctl enable --now auditd`. For persistence across reboots, add rules to `/etc/audit/rules.d/` and reload with `sudo augenrules --load`. For forensics, check `/etc/audit/auditd.conf` for settings like log rotation (`max_log_file`) to understand retention policies.

## Audit Rules

Audit rules are configuration directives used by the Linux Auditing System to specify which system events, such as file operations, system calls, or user actions, should be monitored and logged by auditd. They are typically defined in files like `/etc/audit/audit.rules` or `/etc/audit/rules.d/` and loaded into the kernel. These are used for customizing the scope of auditing to focus on security-critical activities, such as watching specific files for modifications, tracking executable runs, or monitoring network operations, thereby supporting compliance, threat detection, and detailed event tracking without overwhelming the system with unnecessary logs.

There are 3 kinds of audit rules:

- **Control rules** - overall control of the audit system.
- **File System rules (also known as watches)** - audit access to files and directories.
- **Syscall** - audit system calls.

### Audit Rule Types and Parameters

| Rule Type | Parameters | Description |
|---|---|---|
| Control rules | `-b` | sets the maximum amount of existing Audit buffers in the kernel |
| Control rules | `-f` | sets the action that is performed when a critical error is detected |
| Control rules | `-e` | enables and disables the Audit system or locks its configuration |
| Control rules | `-r` | sets the rate of generated messages per second |
| Control rules | `-s` | reports the status of the Audit system |
| Control rules | `-l` | lists all currently loaded Audit rules |
| Control rules | `-D` | deletes all currently loaded Audit rules |
| File System rules | `-r` | read access to a file or a directory. |
| File System rules | `-w` | write access to a file or a directory. |
| File System rules | `-x` | execute access to a file or a directory. |
| File System rules | `-a` | change in the file's or directory's attribute. |
| System call rules | `-a` | action and filter (action can be either always or never. filter specifies which kernel rule-matching filter is applied to the event. The rule-matching filter can be one of the following: task, exit, user, and exclude) |
| System call rules | `-S` | system call (list of system calls can be found in the `/usr/include/asm/unistd_64.h` file |
| System call rules | `-F exe=path_to_exe` | filters events where the executable matches the specified path |

Audit rules operate by being loaded into the kernel's audit subsystem either at boot time or dynamically using the `auditctl` command. When an event matches a rule, it's sent to auditd for logging, with keys (`-k`) for easy searching later. For example, you can detect modification and creation of new users using the following auditd rules:

### Example Audit Rules

| Modifications | Rules |
|---|---|
| User, group, password databases | `w /etc/group -p wa -k etcgroup` |
| User, group, password databases | `w /etc/passwd -p wa -k etcpasswd` |
| User, group, password databases | `w /etc/gshadow -k etcgroup` |
| User, group, password databases | `w /etc/shadow -k etcpasswd` |
| User, group, password databases | `w /etc/security/opasswd -k opasswd` |
| Sudoers file changes | `w /etc/sudoers -p wa -k actions` |
| Sudoers file changes | `w /etc/sudoers.d/ -p wa -k actions` |
| Passwd | `w /usr/bin/passwd -p x -k passwd_modification` |
| Tools to change group identifiers | `w /usr/sbin/groupadd -p x -k group_modification` |
| Tools to change group identifiers | `w /usr/sbin/groupmod -p x -k group_modification` |
| Tools to change group identifiers | `w /usr/sbin/addgroup -p x -k group_modification` |
| Tools to change group identifiers | `w /usr/sbin/useradd -p x -k user_modification` |
| Tools to change group identifiers | `w /usr/sbin/userdel -p x -k user_modification` |
| Tools to change group identifiers | `w /usr/sbin/usermod -p x -k user_modification` |
| Tools to change group identifiers | `w /usr/sbin/adduser -p x -k user_modification` |

Sudoers file modification might be also used for privilege escalation.

## aureport

The aureport utility generates summary reports from audit log files, aggregating data into readable formats like event counts or user activity summaries. By default, all audit.log files in the `/var/log/audit/` directory are queried to create the report. You can specify a different file to run the report against using the `aureport` options `-if file_name` command. Aureport works by processing the audit log file (default `/var/log/audit/audit.log`) and using options to specify report types, such as `-a` for authentication, `-f` for files, or `-u` for users (e.g., `aureport -a` for a summary of authentication events). It can filter by time (`-ts`/`-te`), interpret data (`-i`), and output in formats suitable for further analysis, drawing from hardcoded event categories to produce aggregated statistics like success/failure counts.

### Example Command Usage

```shellsession
linuxforensics@ubuntu:~$ sudo aureport --input /home/linuxforensics/Desktop/cases/scenario1/collection/uploads/auto/var/log/audit/audit.log --login
```

<img width="674" height="214" alt="image" src="https://github.com/user-attachments/assets/b6de743d-e4bc-4f45-95aa-2e357ec54f4c" />

## ausearch

Ausearch is used for investigating audit trails by searching logs for particular events, such as failed logins or file accesses, which is essential for security analysis, debugging, and reconstructing sequences of actions during incident response. By default, ausearch searches the `/var/log/audit/audit.log` file. You can specify a different file using the `ausearch` options `-if file_name` command. Supplying multiple options in one ausearch command is equivalent to using the AND operator between field types and the OR operator between multiple instances of the same field type.

Such a command could look like this:

```shellsession
linuxforensics@ubuntu:~$ sudo ausearch -if 
```

<img width="664" height="213" alt="image" src="https://github.com/user-attachments/assets/3c1406a6-ffba-4ae4-85f8-c0f1fdda4e50" />

To list all ausearch options, we can use `man ausearch`. The full Auditd documentation can be found here.

Audit logs are text-based but structured, with fields like `type=`, `msg=audit(timestamp:event_id):`, and key-value pairs (e.g., `pid=`, `uid=`). This format aids parsing; use scripts or tools like awk/grep for extraction. For advanced analysis, integrate with ELK Stack by forwarding logs via audispd plugins.

## Extending Auditd Monitoring

Extend monitoring with rules for network changes:

```shellsession
linuxforensics@ubuntu:~$ auditctl -a always,exit -F arch=b64 -S socket -k network_mod
```

In investigations, make logs immutable with `chattr +i /var/log/audit/audit.log` to prevent tampering. Correlate with journalctl or syslog for a fuller picture—e.g., match Auditd's syscall events with journal timestamps.

### Checking Auditd Status

For example, to status check:

```shellsession
linuxforensics@ubuntu:~$ sudo auditctl -s
```

<img width="664" height="214" alt="image" src="https://github.com/user-attachments/assets/b2d37fe3-ad63-4c30-93fd-c17f5d07b1e4" />

In the next sections we will use Auditd more in depth and see what information we can retrieve and investigate from the logs in different use cases.

## 🔑 Key Takeaway

Auditd provides detailed, kernel-level auditing of security-relevant system events through configurable rules. The `auditctl` utility manages rules, while `aureport` summarizes audit activity and `ausearch` searches for specific events. By monitoring sensitive files, user and group modifications, executable activity, and system calls, investigators can identify suspicious changes and reconstruct potential attack paths.
