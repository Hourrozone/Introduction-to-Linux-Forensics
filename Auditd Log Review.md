# Auditd Log Review

## Introduction

Timelines allow us to reconstruct system events in chronological order. Creating timelines is crucial because it transforms raw, scattered data into a coherent narrative which allows us to understand what, when happened, and what was triggered certain event. Besides that, a timeline is mandatory since it provides verifiable timestamps for accountability and aids rapid triage (e.g., scoping breach duration) and holds up in court by establishing causality with UTC-aligned, tamper-evident data.

## Zircolite

Previously, auditd was installed on the system, and we will now utilize it to address questions that were not covered in our earlier investigation. To simplify our analysis, we can merge `audit.log.1` and `audit.log` into a single file and use the Zircolite utility to analyze the audit log file.

<img width="744" height="151" alt="image" src="https://github.com/user-attachments/assets/edf0993f-3508-4f60-881c-bdde771f7bc6" />

Next let's run the Zircolite utility:

<img width="671" height="796" alt="image" src="https://github.com/user-attachments/assets/37e410d1-9fe1-49ca-9e56-cb95150efbca" />

What hasn’t been noticed yet so far is user creation, so let’s grep user creation:

<img width="685" height="310" alt="image" src="https://github.com/user-attachments/assets/5368d69c-dbe3-4384-bc1f-ef574064e213" />

Zircolite couldn’t parse the created user. Then let’s use the `aureport` and `ausearch` utilities to parse audit log file.

<img width="622" height="661" alt="image" src="https://github.com/user-attachments/assets/f8a20572-cf14-4c07-a4d2-833b2e9c1b19" />
### How did /bin/bash start at 18:48:25?

The first question is:

Can we answer how `/bin/bash` started at 18:48:25?

We can find PID 28644 started from Parent Process ID (PPID) 916 (apache):

<img width="766" height="799" alt="image" src="https://github.com/user-attachments/assets/98ecce86-4de1-44bc-a399-feb3518fd2a0" />

Next, can we determine the PPID of the `/bin/bash` process that was initiated at 18:48:25?

```shellsession
916
```

### How and which new user was created?

The next question can be:

How and which new user was created?

Therefore, let’s find events associated with user creation:

<img width="741" height="330" alt="image" src="https://github.com/user-attachments/assets/8cf30a28-facb-444f-a471-bfa81c0833a0" />

Alternatively, we can use `aureport` as well to find the username created during an attack:

<img width="646" height="196" alt="image" src="https://github.com/user-attachments/assets/6314dcf3-1a9a-4da0-bfbf-67a20db9ebff" />

In the `Linux.Sys.Users.json` and `Linux.Users.RootUsers.json` we can find the entries for standard and root users:

<img width="740" height="149" alt="image" src="https://github.com/user-attachments/assets/0582c7f9-ba41-42b4-b32d-75fc4aa03356" />

### How was /etc/sudoers modified?

To address the lingering question of how the `/etc/sudoers` file was modified, the following command should provide the necessary information.

<img width="744" height="550" alt="image" src="https://github.com/user-attachments/assets/b7f82030-dee3-48a2-96ca-512b0ed6d6df" />

So we can see that the user used vim to modify the `/etc/sudoers` file and perform privilege escalation. Ausearch can also be used to generate a timeline of events:

<img width="740" height="622" alt="image" src="https://github.com/user-attachments/assets/ed640211-46e2-4dad-9236-222099e070da" />

<img width="762" height="374" alt="image" src="https://github.com/user-attachments/assets/078e1c94-ab20-4549-9741-84a9f3b622cf" />

## Timeline

| Time | Description | User |
|---|---|---|
| 18:03:06 | First (unsuccessful) RCE request | user |
| 18:08:24 | Successful RCE | user |
| 18:15:23 | Linpeas downloaded and executed | user |
| 18:20:50 | `/etc/sudoers` exploited for privilege escalation | user |
| 18:25:18 | user john was created | root |
| 18:27:27 | openssh-server was installed on system | root |
| 18:32:40 | authorized_key added for john user | |
| 18:34:00 | User john authorized using SSH | john |
| 18:35:44 | crontab persistance added | john |
| 18:37:55 | User john disconnected | john |
| 18:48:25 | One more time RCE executed | user |

Retrieving all those timestamps we can create a solid timeline of events and actions that describe when, what, and by whom a certain event was triggered. It makes it possible for us to pinpoint entry vector, trace lateral movements, and quantify the radius of an attack with great precision.

## 🔑 Key Takeaway

Auditd logs provide detailed evidence of process execution, user creation, privilege escalation, and system modifications. Using Zircolite, `aureport`, and `ausearch` helps extract and correlate these events into a chronological timeline. In this investigation, the timeline reveals successful RCE through Apache, LinPEAS execution, modification of `/etc/sudoers`, creation of a new user, SSH persistence, and a cron job configured to repeatedly execute the malicious payload.
