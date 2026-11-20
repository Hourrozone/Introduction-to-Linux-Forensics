# Linux Persistence

## 📖 Introduction to Linux Persistence

Linux persistence techniques refer to methods used by threats, such as malware or attackers, to maintain long-term access to a compromised Linux system. These mechanisms ensure that malicious code or access survives events like reboots, logouts, or system updates.

In a forensics context, understanding these helps investigators identify artifacts, reconstruct incidents, and detect anomalies through logs, file modifications, and process behaviors. Techniques often align with the **MITRE ATT&CK framework's Persistence tactic (TA0003)**, which defines ways adversaries retain footholds via access, actions, or configurations.

Common categories include boot/logon scripts, scheduled tasks, service modifications, user account manipulations, and event-triggered executions.

Below is an outline of key techniques in a table format, drawing from established sources for high-level descriptions and detection approaches. Variations exist across distributions (e.g., Ubuntu vs. CentOS), but many leverage standard components like systemd, cron, or shell configs.

## 🗺️ Linux Persistence Map

Let’s take a look at the Linux Persistence Map:

<img width="3125" height="2959" alt="image" src="https://github.com/user-attachments/assets/38d0768d-1bbd-4962-bc7c-f4ac46cf3c74" />

**Image source**

It illustrates various mechanisms for maintaining access in Linux environments, particularly those using systemd as the init system. It begins at the central node representing the system init process and branches out into interconnected components covering system-wide and user-specific elements, such as generators, services, schedulers, daemons, accounts, authentication methods, and login shells, with additional nodes for specialized areas like web servers, rootkits, and infected software.

This diagram is especially useful to hunt down such persistence mechanisms that are used in the wild.

## 🔍 Common Linux Persistence Mechanisms

Let’s go through some of the common Linux persistence mechanisms:

### 1. User Manipulation

Users and groups configurations are located here:

| `/etc/gshadow` | `/etc/shadow` | `/etc/passwd` | `/etc/group` |
|---|---|---|---|

```shellsession
linuxforensics@ubuntu:~$ cat /etc/passwd
```

<img width="800" height="585" alt="image" src="https://github.com/user-attachments/assets/d9d9e116-f297-477c-84a9-e6981a94ce91" />

Attackers might add new users, modify passwords, or elevate privileges by editing these files. For instance, a new entry in `/etc/passwd` could indicate a backdoor account. In forensics, compare timestamps and contents against baselines to spot anomalies.

### 2. Init Scripts (SysV Init or Upstart)

Init Scripts (aka. SysV Init Scripts) are configuration files that are used by Linux initialization systems to manage the startup, shutdown, and supervision of services and processes during the boot process. Those startup scripts are located in `/etc/init.d/` and symbolic links in various runlevel directories like `/etc/rc3.d/` to start or stop services.

Upstart is used in some older Linux distributions like Ubuntu 14.04 and it's configuration files are located in `/etc/init/`.

### 3. Systemd

Systemd is the modern init system used in most Linux distributions. It manages services using unit files, usually located in `/lib/systemd/system/` or `/etc/systemd/system/`. You can use `systemctl` to enable and start services.

### 4. Cron Jobs

You can use cron jobs to schedule tasks at specific times or intervals. Entries in crontab files (e.g., `/etc/crontab`, or per-user crontab using `crontab -e`) can be used to start scripts or commands.

For example, you can retrieve crontab data using `crontab -l`:

```shellsession
linuxforensics@ubuntu:~$ crontab -l

0 2 * * /bin/sh backup.sh
```

### 5. SSH Authorized_keys

The authorized_keys file, located at `~/.ssh/.authorized_keys`, specifies the SSH keys that can be used to log in to the user account for which the file is configured.

### 6. Startup Applications

Many Linux desktop environments, such as GNOME and KDE, allow users to configure startup applications through graphical settings. These are typically user-specific and can be configured through the desktop's settings.

### 7. `/etc/rc.local`

This is a traditional method for running custom scripts at boot time. The `/etc/rc.local` script is executed at the end of the system's initialization.

### 8. Shell Profile Files

You can add commands to shell profile files like `~/.bashrc`, `~/.bash_profile`, or `~/.profile`. These commands will be executed whenever a user logs in.

### 9. Autostart Directories

Some desktop environments use autostart directories to launch user-specific applications. For example, in GNOME, you can place `.desktop` files in `~/.config/autostart/`.

### 10. Service Management Tools

Tools like `chkconfig` or `update-rc.d` (for SysV Init) can be used to manage services and their runlevel configurations.

### 11. User Session Startup Scripts

In some cases, you may want to execute scripts or programs at the start of a user session. You can achieve this by adding commands to the appropriate shell startup files, such as `~/.xprofile`, or by using the desktop environment's session manager.

## 🔑 Key Takeaway

Linux persistence mechanisms allow attackers to maintain access across reboots, logouts, and system updates. Common locations to investigate include user and group configuration files, init scripts, systemd unit files, cron jobs, SSH authorized keys, startup applications, shell profile files, and user session startup scripts. Examining these artifacts, their contents, and timestamps can help forensic investigators identify unauthorized changes and suspicious persistence mechanisms.
