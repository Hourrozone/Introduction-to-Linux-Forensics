# Linux Forensics Artifacts

## 📖 Introduction to Linux Forensics Artifacts

Linux forensic artifacts refer to digital traces, logs, files, and system data left behind on a Linux operating system that can be analyzed during investigations to reconstruct events, user activities, or security incidents. These artifacts help us to understand system usage, potential breaches, or malicious actions without providing step-by-step guides on exploitation. Common artifacts are found in logs, user directories, configuration files, and system processes.

Ubuntu, like most Linux distributions, follows the **Filesystem Hierarchy Standard (FHS)**, which defines a structured layout for directories and files to ensure consistency across Unix-like systems. This hierarchical tree starts at the root directory (`/`) and organizes everything in a logical manner, with subdirectories serving specific purposes such as system binaries, user data, configurations, and logs.

When conducting a forensic analysis of a Linux system, it's crucial to have a good understanding of the file and directory arrangement on the disk. This knowledge helps investigators efficiently locate important areas and evidence while disregarding less relevant sections.

## 📂 Linux Directories

<img width="3125" height="1667" alt="image" src="https://github.com/user-attachments/assets/aa9aa3d9-97aa-473c-bdf4-a98cd2e0a5f4" />

<text>Source: Practical Linux Forensics book</text>

### `/boot/` and `/efi/`

The `/boot/` and `/efi/` directories store essential files required for booting the system. This includes boot configuration files, kernel parameters, and more. Within these directories, you can also locate both the current and previous kernel versions, along with the initial ramfs, all of which can be subject to examination.

### `/etc/`

The `/etc/` directory traditionally serves as the central repository for system-wide configuration files and related data. Most of these files are easily accessible in plaintext. Configuration files often come with associated directories denoted by a `.d` extension, allowing for the inclusion of additional configuration snippets. It's important to note that user-specific configuration files located in a user's `/home/` directory can take precedence over system-wide configurations in `/etc/`.

Here are some of the main forensics artifacts you might find in the `/etc/` directory:

| Purpose | Path | Description |
|---|---|---|
| User Accounts and Passwords | `/etc/passwd` | Contains user account information |
| User Accounts and Passwords | `/etc/shadow` | Stores password hashes (if shadow password system is used). |
| Group Information | `/etc/group` | Contains information about user groups. |
| System Configuration | `/etc/hostname` | Hostname of the system. |
| System Configuration | `/etc/hosts` | Local DNS resolution. |
| System Configuration | `/etc/issue` | Text displayed before login. |
| System Configuration | `/etc/issue.net` | Text displayed before remote login. |
| System Configuration | `/etc/os-release` | Information about the operating system. |
| Network Configuration | `/etc/network/`, `/etc/netplan/`, or `/etc/NetworkManager/` | Contains network configuration files |
| Network Configuration | `/etc/iptables/`, `/etc/iptables.rules` | Stores firewall rules (if used). |
| Service and Software Configuration | `/etc/services` | Lists network services and their associated ports. |
| Service and Software Configuration | `/etc/cron.d/`, `/etc/cron.daily/` | Configuration for scheduled tasks (cron jobs). |
| Service and Software Configuration | `/etc/sudoers` | Configuration for sudo access. |
| System Logs | `/etc/rsyslog.conf` or `/etc/syslog-ng/syslog-ng.conf` | Configuration for system logging. |
| System Logs | `/etc/logrotate.conf` | Log rotation configuration. |
| Security Configuration | `/etc/security/` | Configuration files related to system security. |
| Security Configuration | `/etc/ssh/sshd_config` | SSH server configuration. |
| Security Configuration | `/etc/pam.d/` | Pluggable Authentication Modules configuration. |
| Package Management | `/etc/apt/` | Configuration for the APT package manager (Debian/Ubuntu). |
| Package Management | `/etc/yum.conf` and `/etc/yum.repos.d/` | Configuration for the YUM package manager (RHEL/CentOS). |
| Hostname Resolution | `/etc/resolv.conf` | DNS resolver configuration. |
| Web Server Configuration | `/etc/apache2/`, `/etc/nginx/` | Standard web server configuration directories. |
| Database Configuration | `/etc/mysql/` | Configuration files for MySQL or MariaDB. |
| Database Configuration | `/etc/postgresql/` | Configuration for PostgreSQL. |
| Shell Configuration | `~/.bashrc`, `~/.bash_profile`, `~/.zshrc`, `/etc` | User-specific shell configuration files can be found in the user's home directory while global shell configurations in `/etc`. |
| User-Specific Configuration | `/etc/skel/` | Contains default files and directories for new user accounts. |

### `/tmp/`

The `/tmp/` directory is designated for the temporary storage of files. In certain Linux distributions, the contents of `/tmp/` might be stored in the system's RAM using the tmpfs virtual memory file system. In forensic images, systems employing tmpfs to mount `/tmp/` will probably appear empty.

### `/run/`

The `/run/` directory is a tmpfs-mounted directory residing in RAM and will likely be empty on a forensic image. On a running system, this directory contains runtime information like PID and lock files, systemd runtime configuration, and more. There may be references to files and directories in `/run/` found in logs or configuration files.

### `/home/` and `/root/`

The `/home/` directory is the default location for user home directories. A user’s home directory contains files the user created or downloaded, including configuration, cache, data, documents, media, desktop contents, and other files the user owns. The root user’s home directory is typically `/root/` of the root filesystem.

These home directories are of significant interest to forensic investigators because they provide information about a system’s human users. The creation (birth) timestamp of a user’s home directory may indicate when the user account was first added.

Here are the main artifacts you can find in the `/home/` directory:

| Purpose | Path | Description |
|---|---|---|
| User Documents and Files | `/home/<username>` | The user's personal files, including documents, images, videos, and other data, are typically stored in subdirectories. |
| Browsing Artifacts | `~/.mozilla/` and `~/.config/google-chrome/` | Directories of Firefox and Chrome web browsers user profiles. These directories may contain browsing history, bookmarks, and cache files. |
| Browsing Cache Files | `~/.cache` | Temporary local copies of web resources like images, scripts, and stylesheets that browsers store to speed up page loading on future visits. |
| Hidden Directories | `~/.<directory-name>` like `~/.config`, `~/.ssh` | Many configuration files and directories start with a dot and these often contain settings and preferences for various applications. |
| Desktop Environment Settings | `~/.config/` | User-specific settings for desktop environments (e.g., GNOME, KDE) can be found in this directory. |
| Shell History | `~/.bash_history` | The command history for a user's shell (e.g., Bash) is stored in this file or a similar location. |
| Email clients | `~/.thunderbird/` | Thunderbird email client data storage location. |
| Messaging application | `~/.config` | Messaging data may be stored in this directory or application-specific directories. |
| Desktop Environment Logs | `~/.local/share` or `~/.config` | Logs and usage data for the user's desktop environment can be found in these directories. |
| Access and Modification Timestamps | `/home/` | Timestamps associated with files and directories in the home directory can indicate when files were created, modified, or accessed. These timestamps are inherent to the file system. |
| Recently Used Files | `~/.local/share/recently-used.xbel` | Lists of recently accessed or similar files. |
| SSH and PGP Keys | `~/.ssh/`, `~/.gnupg/` | SSH and PGP keys can be found in these directories. |
| System Logs | `~/.local` | Some system logs and logs from installed applications may be stored in this directory or other hidden directories. |
| Downloaded Files | `/home/<username>/Downloads` | Users often download files to their directory. |
| Cloud Storage Sync | `~/.dropbox` or `~/.config/google-drive-ocamlfuse` | If a user uses cloud storage services like Dropbox or Google Drive, synchronization data and configuration settings can be found in these directories. |

### `/bin/`, `/sbin/`, `/usr/bin/`, and `/usr/sbin/`

The standard locations for executable programs are `/bin/`, `/sbin/`, `/usr/bin/`, and `/usr/sbin/`. These directories were originally intended to separate groups of programs for users, administrators, the boot process, or for separately mounted filesystems. Today, `/bin/` and `/sbin/` are often symlinked to their corresponding directory in `/usr/`, and in some cases, `/bin/`, `/sbin/`, and `/usr/sbin/` are symlinked to a single `/usr/bin/` directory containing all programs.

### `/lib/` and `/usr/lib/`

The `/lib/` directory is generally symlinked to `/usr/lib/` on most Linux systems today. This includes shared library code (also for multiple platforms), kernel modules, support for programming environments (header files), and more. The `/lib/` directory also contains the default configuration files for many software packages.

### `/usr/`

The `/usr/` directory contains the bulk of the system’s static read-only data. This includes binaries, libraries, documentation, and more. Most Linux systems will symlink `/bin/`, `/sbin/`, and `/lib/` to their equivalents in the `/usr/` subdirectory.

Files located here that are not part of any installed package may be of forensic interest because they were added outside the normal software installation process. These might be manually installed files by a user with root access, or unauthorized files placed by a malicious actor.

### `/var/`

The `/var/` directory contains changing system data (variable) and usually persistent across reboots. The subdirectories below `/var/` are especially interesting from a forensics perspective because they contain logs, cache, historical data, persistent temporary files, the mail and printing subsystems, and much more.

The main forensics artifacts of the `/var/` directory are:

| Purpose | Path | Description |
|---|---|---|
| System Logs | `/var/log/` | This directory contains system logs that record various system events and activities, including login attempts, service startups, and system errors. Common log files include `/var/log/auth.log`, `/var/log/syslog`, and `/var/log/messages`. |
| Package Management Logs | `/var/log/dpkg.log` or `/var/log/yum.log` | These logs track package installations, updates, and removals, which can provide information about software changes on the system. |
| Print Spooler Files | `/var/spool/cups/` | The Common Unix Printing System (CUPS) stores print job information in this directory, potentially revealing print job histories. |
| Mail Server Data | `/var/mail/` or `/var/spool/mail/` | Mailboxes and email-related artifacts are often stored here. |
| Database Files | `/var/lib/` | Databases like MySQL or PostgreSQL store their data files and logs. |
| Temporary Files | `/var/tmp/` and `/var/run/` | Temporary files and directories can contain artifacts such as unclean shutdown logs or remnants of executed processes. |
| Cron and Scheduled Tasks | `/var/spool/cron/crontabs/` | Information about scheduled tasks can be found in these files, which may include scripts and commands executed by cron jobs. |
| Web Server Logs | `/var/log/apache2/` or `/var/log/nginx/` | If a web server is installed, access and error logs can provide insights into web activity. |
| Printer Logs | `/var/log/cups/` | Printer-related logs. |
| DHCP and Network Logs | `/var/log/` | DHCP client and server logs. |
| Security and Authentication Data | `/var/log/secure` or `/var/log/auth.log` | These logs record authentication and security-related events, including login attempts and SSH connections. |
| Package Cache | `/var/cache/apt/archives/` or `/var/cache/yum/` | Package cache may contain downloaded package files, which could be of forensic interest. |
| System State Information | `/var/lib/misc/` | This directory may contain system state information, including files related to networking and hardware. |
| Session Information | `/var/run/utmp` and `/var/log/wtmp` | These files track user login and logout sessions. |
| Kernel Logs | `/var/log/kern.log` or `/var/log/dmesg` | Kernel-related logs. |
| Software Update Information | `/var/lib/update-notifier/package-data-downloads/` | This directory may contain data related to software updates. |

### `/dev/`, `/sys/`, and `/proc/`

Linux has several other tmpfs and pseudo-filesystems that appear to contain files when the system is running, which include `/dev/`, `/sys/`, and `/proc/`. These directories provide representations of devices or kernel data structures but the contents don’t actually exist on a normal filesystem. When examining a forensic image, these directories will likely be empty. See the `procfs(5)` and `sysfs(5)` man pages for more details.

### `/media/`

The `/media/` directory is intended to hold dynamically created mount points for mounting external removable storage, such as CD-ROMs or USB drives. When examining a forensic image, this directory will likely be empty. References to `/media/` in logs, filesystem metadata, or other persistent data may provide information about user-attached (mounted) external storage devices.

### `/opt/`

The `/opt/` directory contains add-on packages, which typically are grouped by vendor name or package name. These packages may create a self-contained directory tree to organize their own files (for example, `bin/`, `etc/`, and other common subdirectories).

### `/lost+found/`

A `/lost+found/` directory may exist on the root of every filesystem. If a filesystem repair is run (using the `fsck` command) and a file is found without a parent directory, that file (sometimes called an orphan) is placed in the `/lost+found/` directory, where it can be recovered. Such files don’t have their original names because the directory that contained the filename is unknown or missing.

## 🔗 Additional Resources

You can find an extensive list of forensic artifacts and their respective locations in the following resources:

- https://digitalforensics.ch/linux/
- https://github.com/orlikoski/CyLR
- https://github.com/ForensicArtifacts/artifacts/blob/main/artifacts/data/linux.yaml

## 🔑 Key Takeaway

Linux forensic artifacts are distributed throughout the filesystem, with important evidence commonly found in configuration directories (`/etc/`), user home directories (`/home/` and `/root/`), system logs (`/var/`), and executable locations (`/usr/`, `/bin/`, and `/sbin/`). Understanding the Filesystem Hierarchy Standard (FHS) helps investigators locate relevant evidence, identify suspicious modifications, and reconstruct user and system activity during an investigation.
