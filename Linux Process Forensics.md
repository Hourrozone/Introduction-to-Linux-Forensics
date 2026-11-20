<img width="677" height="175" alt="image" src="https://github.com/user-attachments/assets/1ddcb583-685c-4758-af2d-dd7d2ab88783" /><img width="793" height="678" alt="image" src="https://github.com/user-attachments/assets/18ee1274-99b9-4d02-b118-e13fe34e931d" /><img width="507" height="27" alt="image" src="https://github.com/user-attachments/assets/d097f2eb-3177-4c83-aac7-19246efe52e4" /># Linux Process Forensics

## Apache Log Analysis

Before diving deeper into the processes, let's inspect the Apache logs to identify any suspicious requests. Vulnerable web applications are often used as an entry point for attackers. Filtering for `/bin/sh` and `/bin/bash` could be a good approach.

<img width="534" height="327" alt="image" src="https://github.com/user-attachments/assets/e6e299c7-c7f1-4f57-a9f9-e0b5bac372be" />

Following the initial access check, we see `/bin/sh` and `/bin/bash` were executed.

Beginning at 18:08:24, we see that requests don’t have a response size. It may indicate that the attacker successfully executed the RCE attack.

Please note that Apache logs do not store request bodies, but if the script was executed by CGI, it’s a good idea to check the process's environment variables. We have a suspicious process `bash -i` with PID 28645 executed, so let’s check it:

<img width="644" height="737" alt="image" src="https://github.com/user-attachments/assets/f2a79fbf-a743-4f6e-905d-34c0096422f0" />

`HTTP_USER_AGENT` and `SERVER_SOFTWARE` are noteworthy.

<img width="312" height="56" alt="image" src="https://github.com/user-attachments/assets/1f7b2db8-fa55-4af3-b138-7fd89c0c1556" />

## Processes

We'll begin with the `linux.malfind.Malfind` Volatility plugin (use PID 28344, which was uncovered in the previous section, and was seen connecting to EC2 IP Address).

<img width="746" height="817" alt="image" src="https://github.com/user-attachments/assets/9ddd4020-9ff2-46dd-9ba6-d6d2397111de" />

From there we can also see that PID 28344 marked as suspicious (potentially contains injected code). We can use `linux.psaux.PsAux` to analyze processes and executed commands.

<img width="706" height="640" alt="image" src="https://github.com/user-attachments/assets/c061e3bd-90d3-49c2-8902-6fe193ab83d6" />

The most valuable for investigation processes:

<img width="702" height="364" alt="image" src="https://github.com/user-attachments/assets/fd7cfbcd-9ff0-45d6-931e-590e79fb143d" />

### PID Tree

<img width="702" height="163" alt="image" src="https://github.com/user-attachments/assets/bf9af4ae-2a43-4de3-847d-0f418c30c014" />

PID 28645 stands out as showing an interactive bash session. For the next steps we should also save Apache processes:

<img width="448" height="147" alt="image" src="https://github.com/user-attachments/assets/fdb540ac-05ba-4187-9792-aebcaa575274" />

Additionally, we can also check with the Volatily's `linux.lsof` plugin every process in the memory dump and enumerate the open file descriptors (FDs) that process holds.

<img width="653" height="295" alt="image" src="https://github.com/user-attachments/assets/762c4d05-7089-4159-a99f-fb0a858a48f6" />

In this output, a path that looks like `/tmp/#71` (or any #number/inode) almost always means that this file has been deleted or unlinked from the file system. This is classic malware behavior:

1. Download a payload into `/tmp` (or `/dev/shm`)
2. Execute it
3. Immediately delete the file so that `ls /tmp` shows nothing
4. Continue running from the still-open file descriptor

If take a look at the PIDs that share the exact same deleted file `/tmp/#71`, we will see that:

| PID | Process | Role in the attack chain |
|---|---|---|
| 28332 | cron | Parent cron process |
| 28333 | sh | Shell that executed the one-liner |
| 28334 | run-one | Ensures the command only runs once |
| 28342 | flock | Locking mechanism used by run-one |
| 28343 | python3 | The Python process that downloaded & executed the payload |
| 28344 | 3 (smd) | The final malicious process we originally detected |

The fact that every single process in the chain holds an open handle to the same deleted inode proves they are all part of the same execution path and are operating on the same payload. The interactive bash session (PID 28645) and the Apache processes do not appear in this filtered list, which is also useful information because they are not directly holding the deleted payload.

Next, we can extract the complete command-line arguments using Volatility’s `linux.psaux` plugin:

<img width="647" height="162" alt="image" src="https://github.com/user-attachments/assets/a60de40d-61ff-497d-a152-0cd2c34d9bde" />

The PID 28343 is a Python process that uses the requests library to download a 2nd stage payload from `http://18.117.8.128:8000/chattingloosened` and then immediately executes that payload by feeding it into another Python interpreter via `subprocess.run`. The child process (PID 28344) that results from this is named `3` and appears as `smd`. The combination provides strong confirmation that we are looking at the malicious payload and the process responsible for launching it.

### Live-System /proc Techniques and Volatility 3 Equivalents

There are several other options to take a closer look at individual processes. The table below shows the live-system `/proc` techniques and their closest Volatility 3 equivalents:

| Live Command / Path | What it shows | Volatility3 Equivalent | Notes / Teaching Tip |
|---|---|---|---|
| `ls -al /proc/<PID>` | Overview of all process artefacts (cwd, exe, fd, maps, status) | Combination of `linux.pslist`, `linux.lsof`, `linux.proc.Maps` | No single plugin replaces the full directory listing. Therefore, use these three together. |
| `cat /proc/<PID>/cmdline` | Full command-line arguments | `linux.psaux.PsAux` | Filter with grep on the PID. |
| `cat /proc/<PID>/comm` | Short process name (≤ 16 characters) | Process / COMM column in `linux.pslist` or `linux.pstree` | Visible in almost every process-listing plugin. |
| `strings /proc/<PID>/environ` | Environment variables | `linux.envars.Envars --pid <PID>` | Direct equivalent. Very useful for understanding launch context. |
| `ls -al /proc/<PID>/fd` | Open file descriptors (including deleted files) | `linux.lsof.Lsof --pid <PID>` | Deleted files appear as `/path/#inode`. |
| `cat /proc/<PID>/stack` | Current kernel stack | Limited (`linux.pscallstack` if available) | Support is incomplete in Volatility3 and often left for live analysis. |
| `cat /proc/<PID>/status` | Detailed process status (UIDs, state, memory, PPID) | Fields in `linux.pslist.PsList` | Most important fields (PID, PPID, UIDs, start time) are already present. |

## Dump processes

So, based on the investigation we can create a list of the most interesting processes:

<img width="699" height="561" alt="image" src="https://github.com/user-attachments/assets/8a3e7372-8bb7-4861-b129-0f57af595f7a" />

### PID list

<img width="507" height="27" alt="image" src="https://github.com/user-attachments/assets/4a0b1912-c6af-4529-b674-5faed52f66a8" />

We can dump those processes using the `linux.proc.Maps` Volatility plugin. Dump Apache processes first (dump them one by one, because Volatility sometimes crashes on this step, it uses too much RAM):

<img width="677" height="363" alt="image" src="https://github.com/user-attachments/assets/41465902-9406-4d40-a1a9-2119418ab5f2" />

After, we can dump process maps related with malicious Python scripts:

<img width="694" height="844" alt="image" src="https://github.com/user-attachments/assets/113cb7e9-f588-4c03-99e9-e510c0d69100" />

## Apache Process Memory Maps

We can try straightforward checks by grepping IoCs that we already know (`bash -i`) in dumped processes:

<img width="500" height="118" alt="image" src="https://github.com/user-attachments/assets/06db530b-7df8-4710-995c-57c0f107866c" />

We can see 2 memory segments related to Apache processes.

Let’s run strings on each of them:

<img width="793" height="678" alt="image" src="https://github.com/user-attachments/assets/54a3b6d2-73e8-4fce-88e2-8c87f82d5312" />

We have already seen similar information in environment variables. Valuable information we can extract from here:

<img width="740" height="249" alt="image" src="https://github.com/user-attachments/assets/c02601f3-21f2-4fb5-8b4e-6fe69fbc0303" />

Next, we can review PID 918:

<img width="798" height="811" alt="image" src="https://github.com/user-attachments/assets/9f6be947-03f1-4899-88e8-325bdc4cb7e9" />

Valuable information:

<img width="783" height="114" alt="image" src="https://github.com/user-attachments/assets/9d370a60-1993-4bf0-b7ab-39870418af17" />

We can check the Apache version for known RCE CVEs and find the CVE that was used by the attacker:

https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-41773

## Miner Executable Memory Maps

Next, let’s try to find the remote IP address:

<img width="589" height="200" alt="image" src="https://github.com/user-attachments/assets/7bd57c3e-5e21-41bf-afd1-47238a9ab368" />

Let's take a look at the pid.28344.vma.0x7f338c675000-0x7f338c6c3000.dmp strings:

<img width="687" height="716" alt="image" src="https://github.com/user-attachments/assets/dcd0b53b-18bf-4348-94c8-09e7de57de98" />

<img width="733" height="737" alt="image" src="https://github.com/user-attachments/assets/0c0a447f-be78-4226-ab4c-6963a2bd2917" />

<img width="664" height="293" alt="image" src="https://github.com/user-attachments/assets/3ee14cf6-5711-4b47-8142-7733c2e830d6" />

We can see that a cryptominer is running on the system. Extracted valuable text data:

<img width="702" height="402" alt="image" src="https://github.com/user-attachments/assets/8bde8305-0d9f-4e52-8eb1-86a038a0912d" />

We can Google “239daadd5c7d0ac097376c7871f787738826eef1c024729eff870e473b970855” and find that it may be associated with TLS fingerprint:

https://www.reddit.com/r/MoneroOcean/comments/154p6vb/xmrig_config_for_ghostrider/

### Login Data Extracted

<img width="674" height="179" alt="image" src="https://github.com/user-attachments/assets/d28a652d-e7cb-43f5-aa10-7ffc071cfb22" />

The agent is identified as XMRig, which is the most commonly used crypto mining software by malicious actors on Linux systems. Next, let's take a look at the pid.28344.vma.0x8f3000-0xa64000.dmp strings:

<img width="647" height="686" alt="image" src="https://github.com/user-attachments/assets/b116e8f5-bb51-4e81-a051-7414da98ce18" />

<img width="629" height="713" alt="image" src="https://github.com/user-attachments/assets/21d5b986-3134-4b7a-b038-85d632ef0d3e" />

<img width="636" height="756" alt="image" src="https://github.com/user-attachments/assets/f1da7790-fe9d-475c-9118-73fb69b93e13" />

## Python Script Memory Maps

Python scripts normally contains import information, let’s grep it:

<img width="482" height="265" alt="image" src="https://github.com/user-attachments/assets/3c5976d7-dba1-4b30-97c4-7d1cafc520b6" />

It looks like pid.28343.vma.0x7f2e104b3000-0x7f2e10b18000.dmp is the file that we are looking for:

<img width="808" height="170" alt="image" src="https://github.com/user-attachments/assets/326f336f-d3f0-4066-9633-a5ffd2153dc2" />

Save script content to `malicious_script.py`:

<img width="729" height="49" alt="image" src="https://github.com/user-attachments/assets/fbbf2f57-ac67-4948-a153-896c52f60aed" />

### Malicious script content

<img width="676" height="247" alt="image" src="https://github.com/user-attachments/assets/dda724b3-fbd6-49a6-8600-b879681f9850" />

After decompressing the base64-encoded text, malware invokes syscall number 319 with arguments that match `memfd_create` and executes it with the process name `smd`.

The memory file descriptor, `memfd`, is a Linux feature that allows the creation of anonymous memory-backed file objects that can be used for various purposes, such as inter-process communication or temporary storage. Threat actors sometimes abuse this Linux feature to execute payloads without writing them to disk, and thus avoid traditional security tools that rely on basic binary scans. Once the payload is placed within a memory section created via `memfd`, attackers can invoke one of the exec syscalls on that memory content, treating it as if it were a regular file on disk, and thereby launch a new process.

Live processes executed from memfd can be identified on an up-and-running workload by inspecting the symbolic link of `/proc/<pid>/exe`, which begins with the `/memfd:` prefix.

While we extracted the Python script with such an easy method, this sample wasn’t extracted fully. Real sample looks like this:

https://www.virustotal.com/gui/file/25232290fa9fa5529240a4e893ce206dfdcfc28d0b3a1b89389f7270f1046

<img width="1600" height="617" alt="image" src="https://github.com/user-attachments/assets/d156a53b-eaac-4f5d-af3f-897b6f739ca7" />

## 🔑 Key Takeaway

The investigation links suspicious Apache requests to an interactive Bash session and a cron-launched Python downloader. Volatility process analysis reveals a chain of processes sharing a deleted payload, while memory-map extraction uncovers the XMRig cryptominer and its configuration data. The Python script also abuses `memfd_create` to execute a payload directly from memory, helping the attacker avoid leaving a conventional executable on disk. Together, Apache logs, process relationships, open file descriptors, environment variables, and memory contents help reconstruct the attack chain and identify the exploitation and execution techniques.
