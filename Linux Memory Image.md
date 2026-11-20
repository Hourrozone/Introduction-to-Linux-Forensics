# Linux Memory Image

## Introduction

Acquiring and analyzing memory images is a crucial technique for capturing volatile data that disappears on shutdown. This process snapshots the system's RAM, revealing active processes, network connections, and even hidden malware that disk-based analysis might miss. Building on tools like SysmonForLinux, memory forensics complements log reviews by providing real-time system state insights, crucial for detecting rootkits or in-memory attacks during incident response.

## Overview of Linux Memory Acquisition

Memory dumps can expose loaded modules, open files, and command histories that aren't persisted to disk, making them essential for volatile evidence preservation. To maximize the effectiveness of Volatility 3 (a forensics tool that we will use in the next section) during our analysis and investigations in real-world environments, we need to acquire memory dumps as early as possible to minimize volatility loss because the data in RAM can evaporate quickly due to system activity or power cycles.

You can use a variety of tools to acquire Linux memory:

- **Fmem** - this is a kernel module which creates a `/dev/fmem` device, that can be used for dumping physical memory, without limits of `/dev/mem` (1MB/1GB).

- **LiME** - this is another Loadable Kernel Module (LKM) which is used to collect volatile memory images from Linux and Linux-based devices, such as Android.

- **AVML** - AVML is an X86_64 userland volatile memory acquisition tool written in Rust, intended to be deployed as a static binary. AVML can be used to acquire memory without knowing the target OS distribution or kernel a priori. No on-target compilation or fingerprinting is needed.

```shellsession
@htb[/htb]$ ./avml --help
```

<img width="559" height="865" alt="image" src="https://github.com/user-attachments/assets/373d8d2a-4608-4f56-8160-9ff942ec2f74" />

## Preparing for Memory Acquisition

Before starting the acquisition process, ensure that the target system is isolated to prevent further changes to the memory state. This might involve disconnecting network cables or disabling wireless interfaces to avoid remote interference. Also, document the system's current state, including running processes via `ps aux` or network connections with `netstat -tuln`, as these can serve as baselines for later analysis. Remember, inserting any tool into the system can alter memory slightly, so choose tools that minimize footprint, like static binaries.

## Memory Dump with AVML

The easiest tool to use is AVML, which doesn’t require any compilation on the target system. To acquire a memory image, just run it with root permissions and provide the filename.

```shellsession
@htb[/htb]$ sudo ./avml memdump.mem

Acquiring memory...
Progress: 100% (4.0 GB / 4.0 GB)
Memory acquisition complete. File saved as memdump.mem
```

## Using LiME for Memory Dumps

For environments where kernel modules are acceptable, LiME offers flexible options for dumping memory over TCP or to local files. First, compile the module on a similar system (matching kernel version) using make, then transfer the `.ko` file to the target. Load it with insmod and specify the output format.

```shellsession
@htb[/htb]$ sudo insmod lime.ko "path=tcp:4444 format=lime"

# On the collection host:
@htb[/htb]$ nc -l -p 4444 > memory.lime
```

This method is useful for remote acquisitions without writing to the target's disk, reducing the risk of overwriting evidence.

## Considerations

Always verify the integrity of the acquired image using hashes like SHA-256 immediately after dumping to ensure no corruption occurred during transfer. Store the dump on external media to avoid contaminating the target system. Be aware of legal implications in forensic scenarios, as the chain of custody must always be maintained. In virtualized environments, consider hypervisor-level dumps if possible for a cleaner snapshot. Finally, practice these techniques in a lab setting to familiarize yourself with potential errors, such as module loading failures due to secure boot restrictions.

## 🔑 Key Takeaway

Linux memory acquisition preserves volatile evidence that may disappear when a system shuts down or changes state. Tools like AVML, LiME, and Fmem can capture RAM for later analysis with Volatility 3. Investigators should acquire memory early, minimize changes to the target, verify image integrity with hashes, and maintain a clear chain of custody.
