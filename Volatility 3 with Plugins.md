# Volatility 3 with Plugins

## Introduction

Volatility 3 is an advanced, open-source memory forensics framework designed to extract and analyze digital artifacts from volatile memory dumps, supporting multiple operating systems, including Linux. It represents a complete rewrite of the original Volatility tool, offering improved performance, a more modular architecture, and better handling of modern systems without relying on pre-built profiles. Instead, it uses symbol tables for kernel structures and enables us to dissect memory images from systems to uncover hidden processes, network activity, command history, and other indicators of compromise, making it essential for incident response, malware analysis, and forensic investigations. Unlike its predecessor (Volatility 2), Volatility 3 uses a plugin-based system with automated symbol resolution, reducing setup complexity while maintaining compatibility with similar plugin functionalities.

## Setting Up Volatility 3 for Linux Analysis

Volatility 3 is already installed on the target.

However, if you have to get started with Volatility 3 on Linux memory dumps, first install the framework by cloning the repository from GitHub and installing dependencies via pip:

```shellsession
linuxforensics@ubuntu:~$ git clone https://github.com/volatilityfoundation/volatility3.git
linuxforensics@ubuntu:~$ pip install -r requirements.txt
```

For Linux-specific analysis, you'll need a memory dump file (e.g., `memdump.mem`) acquired from the target system. Volatility 3 requires symbol tables for the Linux kernel, which can be generated or downloaded. You can search for the full banner as shown in Volatility 3’s `banners.Banners` plugin or check the kernel version using `uname -r`. To obtain the appropriate Linux symbols, visit the community-maintained repository at https://github.com/Abyss-W4tcher/volatility3-symbols, which provides pre-generated JSON symbol packs for various Linux kernel versions and distributions.

Pre-built symbol tables are often available, but for custom kernels, use tools like `dwarfdump` to create them from kernel debug symbols (`vmlinux` file) and `System.map`. Place symbol tables in a directory and point Volatility to them using the `--symbol-dir` option if needed. We can run plugins with the command format `python3 vol.py -f <dumpfile> <plugin_name>`, ensuring the Python environment matches the framework's requirements. Test setup by running `python3 vol.py --info` to list available plugins and automations.

## Key Volatility 3 Plugins for Linux

Once configured, Volatility 3 offers a suite of Linux-specific plugins to extract valuable forensic data from memory dumps. These plugins leverage the framework's ability to parse kernel structures like `task_struct` for processes or inode caches for files. Below is an expanded list of commonly used plugins, including those mentioned in the original description, with brief explanations, usage commands, and typical outputs:

| Plugin | Description | Command |
|---|---|---|
| `linux.pslist` | Lists all running processes by walking the kernel's task list, displaying details like PID, PPID, command name, and start time. Useful for identifying hidden or rogue processes. | `python3 vol.py -f memdump.mem linux.pslist` |
| `linux.bash` | Recovers command history from bash shells by scanning process memory for history structures, revealing executed commands even if logs were cleared. | `python3 vol.py -f memdump.mem linux.bash` |
| `linux.sockstat` | Lists all network connections for all processes, including sockets, protocols (TCP/UDP), local/remote addresses, and associated processes. Ideal for detecting unauthorized network activity or backdoors. | `python3 vol.py -f memdump.mem linux.sockstat` |
| `linux.psaux` | Lists processes with their command line arguments. | `python3 vol.py -f memdump.mem linux.psaux` |
| `linux.pstree` | Displays the process tree, showing parent-child relationships to visualize process hierarchies and spot anomalies like orphaned processes. | `python3 vol.py -f memdump.mem linux.pstree` |
| `linux.lsmod` | Lists loaded kernel modules, helping detect rootkits or malicious drivers injected into the kernel. | `python3 vol.py -f memdump.mem linux.lsmod` |
| `linux.lsof` | Shows open files for each process, including file descriptors, types (e.g., regular files, sockets), and paths, useful for tracking file access during incidents. | `python3 vol.py -f memdump.mem linux.lsof` |
| `linux.envvars` | Extracts environment variables from processes, which can reveal configuration details, paths, or sensitive data like API keys. | `python3 vol.py -f memdump.mem linux.envvars` |
| `linux.malfind` | Scans process memory for suspicious code injections or hidden malware by detecting regions with executable permissions but no mapped files. | `python3 vol.py -f memdump.mem linux.malfind` |
| `linux.check_syscall` | Verifies the integrity of system call tables to detect hooks or modifications by rootkits. | `python3 vol.py -f memdump.mem linux.check_syscall` |
| `linux.elfs` | Lists all memory mapped ELF files for all processes. | `python3 vol.py -f memdump.mem linux.elfs` |
| `linux.mountinfo` | Lists mount points on processes mount namespaces. | `python3 vol.py -f memdump.mem linux.mountinfo` |
| `linux.proc` | Lists all memory maps for all processes. | `python3 vol.py -f memdump.mem linux.proc` |

Additional plugins like `linux.ifconfig` (network interfaces), `linux.mount_cache` (mounted filesystems), and `linux.pidhashtable` (process ID hash table enumeration) provide deeper insights into system state. You can chain plugins or use automations for comprehensive analysis, such as combining `linux.pslist` with `linux.malfind` to investigate suspicious processes.

## Memory Analysis with Volatility

Once the memory image is successfully acquired, the next step is memory analysis. This is where tools like Volatility come into play. Volatility is an open-source memory forensics framework that provides a wealth of capabilities for analyzing memory dumps. The only difference between Linux and Windows memory analysis is that for Linux you need to create a custom profile (Volatility 2) or symbol table (Volatility 3). We will focus on creating a symbol table for Volatility 3.

First we need to determine the kernel version of the system where the memory image was acquired. We can use volatility module banners.

<img width="724" height="244" alt="image" src="https://github.com/user-attachments/assets/fde2afaa-b8a7-430b-bce4-3b38b00ad098" />

### Example Usage

Let’s take a look at another example and use the `linux.pslist` plugin on a sample memory dump:

<img width="648" height="241" alt="image" src="https://github.com/user-attachments/assets/6bfedffd-873d-4c45-974f-d024fc2c1f5e" />

This output shows process details, which can be piped to tools like `grep` for filtering (e.g., `| grep sshd` to focus on SSH-related processes). For verbose output, add options like `--pid <PID>` to target specific processes.

### Generating Your Own Symbol Table

To generate your own symbol table, follow the next steps:

1. Install `dwarf2json` - it is a Go utility that processes files containing symbol and type information to generate Volatility 3 Intermediate Symbol File (ISF) JSON output suitable for Linux and macOS analysis. Go version 1.14+ is required to build it.
2. Collect system version either through Volatility 3 banners or `uname -r`. In this case system version is 5.15.0-84-generic.
3. To install debug symbols we need to enable dbgsym repositories first.

<img width="697" height="215" alt="image" src="https://github.com/user-attachments/assets/6eb324be-f36e-40c4-801b-d14dd84c94d2" />

4. Now you can install dbgsym for the desired kernel image version.

<img width="597" height="31" alt="image" src="https://github.com/user-attachments/assets/f0e21442-fb73-4bfd-8e74-ed41cafd8acf" />

 The `vmlinux` file containing kernel debug symbols is located in the `/usr/lib/debug/boot` directory.
 
5. Generate Intermediate Symbol File (ISF) JSON for kernel debug symbols. Processing large DWARF files requires a minimum of 8GB RAM, so you may need to add more swap space if you don’t have enough RAM.

<img width="638" height="50" alt="image" src="https://github.com/user-attachments/assets/e0d97f9a-ca80-475e-aa84-7f3e2844938c" />

6. Copy the generated JSON file to the Volatility 3 symbols folder.

<img width="489" height="46" alt="image" src="https://github.com/user-attachments/assets/efb4596f-2b69-4e09-9caf-80a3abd2e1ae" />

Now we are able to analyze our memory image with Volatility 3.

## Volatility 3 Linux Plugin Examples

### linux.psaux

Lists processes with their command line arguments.

<img width="504" height="396" alt="image" src="https://github.com/user-attachments/assets/756efd67-c99a-46b7-ad85-cf08867de705" />

### linux.bash

Recovers bash command history from memory.

<img width="694" height="316" alt="image" src="https://github.com/user-attachments/assets/7448454d-49c2-41f4-8c4b-9653729d72bf" />

### linux.elfs

Lists all memory mapped ELF files for all processes. You can also dump those files with `--dump` option.

<img width="739" height="619" alt="image" src="https://github.com/user-attachments/assets/8d251799-cf0d-4f83-b850-1d49cc5fbaef" />

### linux.envvars

Lists processes with their environment variables.

<img width="631" height="718" alt="image" src="https://github.com/user-attachments/assets/f76c3847-4e9f-4d54-a80d-681d261d5356" />

### linux.lsof

Lists open files similar to lsof command.

<img width="497" height="684" alt="image" src="https://github.com/user-attachments/assets/c35ecb25-c5e6-4bf2-8f86-cab37f7b2d73" />

### linux.malfind

Lists process memory ranges that potentially contain injected code.

<img width="719" height="799" alt="image" src="https://github.com/user-attachments/assets/4d27f497-7aa8-4f40-b022-243825361331" />

### linux.mountinfo

Lists mount points on processes mount namespaces.

<img width="698" height="385" alt="image" src="https://github.com/user-attachments/assets/96f983c1-74fc-466a-b686-d8ad05e1e999" />

### linux.proc.Maps

Lists all memory maps for all processes.

<img width="683" height="768" alt="image" src="https://github.com/user-attachments/assets/979aa382-a5a5-42c5-8ac7-e3ec5ac35a2d" />

To dump a specific process, use `--pid` and `--dump` options. This command will save each memory segment to a separate file.

<img width="680" height="731" alt="image" src="https://github.com/user-attachments/assets/01ef8d18-e561-4a5d-8de0-07dc42e2f872" />

### linux.sockstat

Lists all network connections for all processes.

<img width="668" height="553" alt="image" src="https://github.com/user-attachments/assets/9e6369bd-7c00-4137-a309-b7ffc0aca019" />

## 🔑 Key Takeaway

Volatility 3 is a plugin-based memory forensics framework that uses Linux kernel symbol tables to analyze memory dumps. Its Linux plugins help investigators examine processes, command history, network connections, environment variables, open files, memory mappings, and potential code injections. Generating the correct ISF symbol table for the target kernel is an essential step in preparing Volatility 3 for Linux memory analysis.
