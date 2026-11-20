# File Modifications

## Introduction

Examining file modifications is a critical step for uncovering evidence of unauthorized activity, data tampering, or system compromises. Linux filesystems maintain several key timestamps for each file, often referred to as MACB times:

- **Modification Time (mtime):** Records the last time the file's content was changed.
- **Access Time (atime):** Tracks when the file was last read or accessed.
- **Change Time (ctime):** Notes changes to the file's metadata, such as permissions or ownership.
- **Birth Time (btime):** Available on certain filesystems like ext4, indicating when the file was created.

These timestamps, stored in the inode structure, can reveal timelines of events during an investigation. However, there are certain techniques like timestomping out there that can make the entire inv estigation more difficult.

Timestomping is a pretty sneaky anti-forensic trick attackers use to mess with file timestamps, making it harder to piece together what happened during an incident. Basically, it's all about changing those MACB times to either hide tracks or plant false trails. This can throw off timeline analysis big time, like making a malicious file look like it was created way before the breach. So just keep in mind that there are some techniques that can mess with the obvious logic of events you see.

## Last modified files review

Keeping this possible anti-forensic technique aside, let's look at the last modified files. Therefore, our next step involves identifying files that were modified during the attack. To achieve this, we can use the find command:

<img width="710" height="95" alt="image" src="https://github.com/user-attachments/assets/ed44492b-0ca7-4f0e-8826-d8950328e908" />

The most interesting files modified during this time:

<img width="759" height="604" alt="image" src="https://github.com/user-attachments/assets/a5f60d92-f1e9-4aea-9d2a-d8f2f175a693" />

### ARP File

<img width="713" height="222" alt="image" src="https://github.com/user-attachments/assets/c7edac7f-1069-44fc-9302-948849552f95" />

We can utilize this MAC address to determine that the local address attacker system was running a virtual machine (VM) using VMware when we visit the following page: https://hwaddress.com/oui-iab/00-0C-29/

<img width="919" height="148" alt="image" src="https://github.com/user-attachments/assets/4de3aee1-afaf-4d05-a519-ff1ab733b742" />

### Sudoers file

We observed multiple times that the `/etc/sudoers` file was modified, so let’s check it:

<img width="789" height="304" alt="image" src="https://github.com/user-attachments/assets/ccaf4f5f-f634-435d-a469-b28d6d477e79" />

Here we can see that the line with `user ALL=(ALL:ALL) NOPASSWD: ALL` was added to allow execution using user "user" without the need of typing a password.

Using stat command, we can check when the `/etc/sudoers` file was modified:

<img width="752" height="273" alt="image" src="https://github.com/user-attachments/assets/06b7d8bb-786d-4bc4-9b06-85d0b3034357" />

**Modification time:** 18:20:50.

However, making modifications to the sudoers file is restricted to users with sudo privileges. How did the user modify this file? We have previously observed executed commands that checked SUID binary attributes and opened the `/etc/sudoers` file using the vim command.

Velociraptor has collected SUID binary attributes. We can validate our hypothesis by examining the `Linux.Sys.SUID.json` file.

<img width="757" height="150" alt="image" src="https://github.com/user-attachments/assets/73235967-7341-4523-b876-d7a998e269a9" />

The string `urwxr-xr-x` does not clearly indicate the presence of the SUID attribute on a file. The mode `urwxr-xr-x` means that the owner has full read, write, and execute permissions, while the group and others have read and execute permissions but no write permissions. Additionally, the `.viminfo` file hasn’t been found for user "user".

### History of installed packages

Another file that has to be checked is `history.log`, which contains information about installed packages:

<img width="731" height="39" alt="image" src="https://github.com/user-attachments/assets/be6a54ab-0078-409e-bc79-e9e77b8ef3e4" />

We can find that the openssh-server package was installed on the system.

<img width="727" height="201" alt="image" src="https://github.com/user-attachments/assets/2c35e3b1-57c8-4da2-acdf-14301b868cb2" />

### SSH Authorized_keys file

An authorized key in SSH is a public key used to facilitate login access for users, and this authentication mechanism is commonly known as public key authentication. Each user typically configures their authorized keys individually, often stored in a file named `.ssh/authorized_keys` located within the user's home directory. This setup allows users to authenticate securely with their public keys instead of relying on traditional password-based authentication.

We can see that SSH authorized_keys for user john was modified.

<img width="752" height="133" alt="image" src="https://github.com/user-attachments/assets/dd31ff11-2310-4fbc-a25f-5ab061a749e0" />

In the `Linux.Ssh.AuthorizedKeys.json` file:

<img width="751" height="117" alt="image" src="https://github.com/user-attachments/assets/06c97bbe-df75-4014-b0a7-704d5bd4db35" />

We can also check root's `.viminfo` file to see when the file was opened for modification:

<img width="542" height="722" alt="image" src="https://github.com/user-attachments/assets/68fbc946-331c-473b-8526-5a9dcce6315f" />

## SSH login history

We can use last binary to read wtmp file:

<img width="690" height="307" alt="image" src="https://github.com/user-attachments/assets/e9568625-8c6a-4830-b61e-707c08945d2b" />

The same information is stored in the `Linux.Sys.LastUserLogin.json` file collected by Velociraptor.

<img width="751" height="113" alt="image" src="https://github.com/user-attachments/assets/d8019bbb-7099-4e3e-95c7-99555bcdfb08" />

Given that SSH was installed on the system, we should proceed to examine the SSHD logs found in the `auth.log` file.

<img width="759" height="58" alt="image" src="https://github.com/user-attachments/assets/3eb3bf44-c8ab-474f-b7c8-bbbc2bd388c1" />

Alternatively review `Linux.Syslog.SSHLogin.json` file:

```shellsession
{"Time":"0000-10-15T18:34:00Z","IP":"192.168.152.180","Result":"Accepted","Method":"publickey","AttemptedUser":"j
```

## 🔑 Key Takeaway

File modification analysis helps reconstruct attacker activity by correlating MACB timestamps, configuration changes, package installation history, SSH authorized keys, and login records. In this investigation, modifications to `/etc/sudoers` suggest a privilege escalation attempt, while changes to SSH authorized keys and SSH login records provide evidence of possible unauthorized access and persistence. Timestomping must also be considered when interpreting file timestamps, as attackers can manipulate them to obscure the actual timeline.
