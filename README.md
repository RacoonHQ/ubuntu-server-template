# Template Konfigurasi Server Ubuntu 24.04 LTS

## License

© 2025 RacoonHQ. All rights reserved.

This work is licensed under the MIT License. You may obtain a copy of the License at:

[LICENSE](./LICENSE)

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

## Deskripsi

Dokumen ini menyajikan panduan lengkap dan terstruktur untuk mengonfigurasi server Ubuntu 24.04 LTS di lingkungan virtual machine (VM) menggunakan VirtualBox. Konfigurasi mencakup jaringan ganda (Host-Only dan NAT), akses SSH dengan port custom, server DNS (BIND9), virtual host Apache, sertifikat SSL self-signed (untuk testing lokal), dan file sharing Samba. Panduan ini dirancang sebagai template yang dapat disesuaikan untuk proyek pengembangan atau testing internal.

## OVA Template

Untuk kemudahan deployment, tersedia file `Ubuntu-Server-Template.ova` yang berisi:

- Ubuntu 24.04 LTS pre-installed
- SSH dengan port custom 2222
- SSL self-signed certificate
- DNS server BIND9 terkonfigurasi
- Samba file sharing
- Apache virtual host
- Jaringan ganda (NAT + Host-Only)

**Download Link:** [📥 Ubuntu-Server-Template.ova](https://drive.google.com/file/d/1xhbjcaMka-jJ4iZ2plrIJz_W4JxofRo2/view?usp=sharing)

Cara penggunaan:
1. Import file `.ova` ke VirtualBox
2. Sesuaikan IP address dan domain sesuai kebutuhan
3. Start VM dan gunakan sesuai panduan di bawah

## Catatan Penting

- Asumsikan akses awal melalui console VM (misalnya, VirtualBox GUI)
- Ganti placeholder seperti `[IP_HOST_ONLY]` (contoh: `192.168.56.10`), `[IP_PUBLIK]` (contoh: `185.125.190.21`), dan `[DOMAIN]` (contoh: `serverubuntu.com`) dengan nilai aktual Anda
- Jalankan semua perintah sebagai pengguna dengan hak sudo
- Pastikan VM memiliki akses internet via NAT untuk instalasi paket

## Prasyarat

- Instal Ubuntu 24.04 LTS ISO di VirtualBox
- Konfigurasi jaringan VM:
  - Adapter 1: NAT (interface `enp0s3` untuk internet)
  - Adapter 2: Host-Only (interface `enp0s8` untuk akses lokal dari host Windows)
- Di host Windows, aktifkan VirtualBox Host-Only Adapter dengan IP `[IP_GATEWAY_HOST_ONLY]` (contoh: `192.168.56.1/24`)
- Unduh PuTTY untuk akses SSH dari host Windows

---

## 1. Konfigurasi Jaringan (Host-Only dan NAT)

Pastikan konektivitas dua arah untuk testing.

### Di VM, edit file Netplan:
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

### Isi dengan konfigurasi berikut:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true  # NAT untuk internet
    enp0s8:
      dhcp4: false
      addresses: [[IP_HOST_ONLY]/24] 
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### Terapkan konfigurasi:
```bash
sudo netplan apply
```

### Verifikasi IP:
```bash
ip addr show enp0s8
ping [IP_GATEWAY_HOST_ONLY]
```

### Di host Windows, test ping:
```bash
ping [IP_HOST_ONLY]
```

---

## 2. Update Sistem dan Instalasi Paket Dasar

### Update dan upgrade sistem:
```bash
sudo apt update && sudo apt upgrade -y
```

### Instal paket utama:
```bash
sudo apt install openssh-server bind9 apache2 certbot python3-certbot-apache samba ufw openssl -y
```

### Set hostname (opsional):
```bash
sudo hostnamectl set-hostname [NAMA_SERVER]
sudo reboot
```

---

## 3. Konfigurasi SSH (Port Custom 2222)

### Aktifkan SSH:
```bash
sudo systemctl enable ssh && sudo systemctl start ssh
```

### Edit konfigurasi SSH:
```bash
sudo nano /etc/ssh/sshd_config
```

### Ubah pengaturan:
```
Port 2222
PermitRootLogin no
```

### Restart SSH:
```bash
sudo systemctl restart ssh
```

### Konfigurasi UFW:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.0/24 to any port 2222 proto tcp
sudo ufw --force enable
sudo ufw deny 22/tcp
sudo ufw reload
```

### Verifikasi: Akses via PuTTY (`[IP_HOST_ONLY]:2222`)

---

## 4. Konfigurasi Samba (File Sharing)

### Buat direktori share:
```bash
sudo mkdir -p /srv/samba/share
sudo chown -R nobody:nogroup /srv/samba/share/
sudo chmod -R 0777 /srv/samba/share/
```

### Edit konfigurasi Samba:
```bash
sudo nano /etc/samba/smb.conf
```

### Tambahkan di akhir file:
```ini
[shared]
path = /srv/samba/share
browseable = yes
writable = yes
guest ok = yes
read only = no
create mask = 0664
directory mask = 0775
```

### Restart Samba:
```bash
sudo systemctl restart smbd && sudo systemctl enable smbd
sudo testparm
```

### Konfigurasi UFW untuk Samba:
```bash
sudo ufw allow from 192.168.56.0/24 to any app Samba && sudo ufw reload
```

### Verifikasi: Di Windows, akses `\\[IP_HOST_ONLY]\shared`

---

## 5. Konfigurasi DNS Server (BIND9) untuk `[DOMAIN]` dan `webmail.[DOMAIN]`

### Edit zona utama:
```bash
sudo nano /etc/bind/named.conf.local
```

### Tambahkan zona:
```bind
zone "[DOMAIN]" {
    type master;
    file "/etc/bind/db.[DOMAIN]";
};
```

### Buat file zona:
```bash
sudo cp /etc/bind/db.serverubuntu.com /etc/bind/db.[DOMAIN]
sudo nano /etc/bind/db.[DOMAIN]
```

### Isi file zona (ganti IP dengan `[IP_PUBLIK]` atau `[IP_HOST_ONLY]`):
```bind
$TTL    604800
@       IN      SOA     ns1.[DOMAIN]. root.[DOMAIN]. (
                     3         ; Serial
                604800         ; Refresh
                 86400         ; Retry
               2419200         ; Expire
                604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.[DOMAIN].
@       IN      A       [IP_PUBLIK]
ns1     IN      A       [IP_PUBLIK]
webmail IN     A       [IP_PUBLIK]
```

### Edit opsi BIND:
```bash
sudo nano /etc/bind/named.conf.options
```

### Tambahkan di options:
```bind
listen-on { [IP_HOST_ONLY]; 127.0.0.1; };
allow-query { localhost; 192.168.56.0/24; };
```

### Validasi dan restart:
```bash
sudo named-checkconf
sudo named-checkzone [DOMAIN] /etc/bind/db.[DOMAIN]
sudo systemctl restart bind9 && sudo systemctl enable bind9
```

### Konfigurasi UFW untuk DNS:
```bash
sudo ufw allow from 192.168.56.0/24 to any port 53 proto udp && sudo ufw reload
```

### Verifikasi:
```bash
dig @localhost [DOMAIN]
```

### Edit hosts di Windows:
```
[IP_PUBLIK] [DOMAIN] webmail.[DOMAIN]
```

---

## 6. Konfigurasi Virtual Host (Apache)

### Buat direktori:
```bash
sudo mkdir -p /var/www/[DOMAIN]/public_html
sudo mkdir -p /var/www/webmail.[DOMAIN]/public_html
sudo chown -R $USER:$USER /var/www/[DOMAIN]/public_html
sudo chown -R $USER:$USER /var/www/webmail.[DOMAIN]/public_html
```

### Buat index.html:
```bash
echo "<h1>Selamat Datang di [DOMAIN]</h1>" | sudo tee /var/www/[DOMAIN]/public_html/index.html
echo "<h1>Webmail [DOMAIN]</h1>" | sudo tee /var/www/webmail.[DOMAIN]/public_html/index.html
```

### Virtual host utama:
```bash
sudo nano /etc/apache2/sites-available/[DOMAIN].conf
```

### Isi konfigurasi:
```apache
<VirtualHost *:80>
    ServerName [DOMAIN]
    DocumentRoot /var/www/[DOMAIN]/public_html
    <Directory /var/www/[DOMAIN]/public_html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/[DOMAIN]_error.log
    CustomLog ${APACHE_LOG_DIR}/[DOMAIN]_access.log combined
</VirtualHost>
```

### Aktifkan virtual host:
```bash
sudo a2ensite [DOMAIN].conf
```

### Virtual host webmail (mirip, ganti ServerName dan DocumentRoot):
```bash
sudo nano /etc/apache2/sites-available/webmail.[DOMAIN].conf
sudo a2ensite webmail.[DOMAIN].conf
```

### Nonaktifkan default dan aktifkan modul:
```bash
sudo a2dissite 000-default.conf
sudo a2enmod ssl rewrite
sudo systemctl restart apache2
```

### Konfigurasi UFW untuk HTTP:
```bash
sudo ufw allow from 192.168.56.0/24 to any port 80 proto tcp && sudo ufw reload
```

### Verifikasi: Akses `http://[DOMAIN]` di browser Windows

---

## 7. Konfigurasi SSL (Self-Signed untuk Testing Lokal)

### Buat sertifikat self-signed:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/[DOMAIN].key -out /etc/ssl/certs/[DOMAIN].crt -subj "/CN=[DOMAIN]"
```

### Edit virtual host untuk HTTPS:
```bash
sudo nano /etc/apache2/sites-available/[DOMAIN].conf
```

### Tambahkan blok SSL:
```apache
<VirtualHost *:443>
    ServerName [DOMAIN]
    DocumentRoot /var/www/[DOMAIN]/public_html
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/[DOMAIN].crt
    SSLCertificateKeyFile /etc/ssl/private/[DOMAIN].key
    <Directory /var/www/[DOMAIN]/public_html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Ulangi untuk webmail, lalu restart Apache:
```bash
sudo systemctl restart apache2
```

### Konfigurasi UFW untuk HTTPS:
```bash
sudo ufw allow from 192.168.56.0/24 to any port 443 proto tcp && sudo ufw reload
```

### Verifikasi: Akses `https://[DOMAIN]` (terima peringatan browser)

---

## Troubleshooting Umum

### Ping gagal
Periksa UFW ICMP:
```bash
sudo nano /etc/ufw/before.rules
# Tambahkan: -A ufw-before-input -s 192.168.56.0/24 -p icmp --icmp-type echo-request -j ACCEPT
sudo ufw reload
```

### BIND error
```bash
sudo named-checkconf
# Periksa tanda titik koma (;)
```

### Apache failed
```bash
sudo apache2ctl configtest
# Periksa sintaks file .conf
```

### Situs tidak tampil
Edit hosts di Windows, flush DNS:
```bash
ipconfig /flushdns
```

### Certbot gagal
Gunakan self-signed untuk lokal; untuk produksi, daftarkan domain dan buka port 80/443.

---

## Kesimpulan

Template ini menyediakan fondasi lengkap untuk server Ubuntu yang aman dan fungsional. Untuk produksi, pertimbangkan:
- Domain publik
- Firewall lebih ketat
- Monitoring (misalnya, Fail2Ban)

Jika memerlukan modifikasi, sesuaikan placeholder dan uji setiap bagian secara bertahap. Hubungi tim dukungan untuk isu spesifik.

---

## Informasi Dokumen

- **Versi Dokumen:** 1.0
- **Tanggal Pembuatan:** 15 November 2025
- **Sistem Target:** Ubuntu 24.04 LTS
- **Virtualisasi:** VirtualBox
- **Host OS:** Windows

---

## Placeholder yang Perlu Diganti

- `[IP_HOST_ONLY]` - IP address untuk interface Host-Only (contoh: `192.168.56.10`)
- `[IP_PUBLIK]` - IP address publik atau eksternal (contoh: `185.125.190.21`)
- `[IP_GATEWAY_HOST_ONLY]` - Gateway untuk Host-Only network (contoh: `192.168.56.1`)
- `[DOMAIN]` - Nama domain utama (contoh: `serverubuntu.com`)
- `[NAMA_SERVER]` - Hostname untuk server (contoh: `ubuntu-server`)
