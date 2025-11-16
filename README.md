# Template Konfigurasi Server Ubuntu 24.04 LTS (Manual Setup)

## License

© 2025 RacoonHQ. All rights reserved.

This work is licensed under the MIT License. You may obtain a copy of the License at:

[LICENSE](./LICENSE)

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF, OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

## Deskripsi

Dokumen ini menyajikan panduan lengkap dan terstruktur untuk mengonfigurasi server Ubuntu 24.04 LTS dari awal di lingkungan virtual machine (VM) menggunakan VirtualBox. Konfigurasi mencakup jaringan ganda (Host-Only dan NAT), akses SSH dengan port custom, server DNS (BIND9), virtual host Apache, sertifikat SSL self-signed (untuk testing lokal), dan file sharing Samba. Panduan ini dirancang sebagai template yang dapat disesuaikan untuk proyek pengembangan atau testing internal.

**🚀 Ingin setup yang lebih cepat?** Lihat [README_TEMPLATE.md](./README_TEMPLATE.md) untuk OVA template dengan semua service pre-configured.

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

## 1. Membuat User Baru dan Konfigurasi Sudo

### Buat user baru:
```bash
sudo adduser [NAMA_USER]
```

### Tambahkan user ke group sudo:
```bash
sudo usermod -aG sudo [NAMA_USER]
```

### Verifikasi user dan group:
```bash
groups [NAMA_USER]
# Harus menampilkan: [NAMA_USER] sudo
```

### Login ke user baru (opsional):
```bash
su - [NAMA_USER]
# Verifikasi sudo dengan: sudo whoami
# Harus menampilkan: root
exit
```

**Catatan Penting:**
- Ganti `[NAMA_USER]` dengan nama user yang diinginkan
- Disarankan untuk tidak menggunakan `root` untuk operasi sehari-hari
- User yang ditambahkan ke group sudo akan memiliki hak akses administrator

---

## 2. Konfigurasi Jaringan (Host-Only dan NAT)

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

## 3. Update Sistem dan Instalasi Paket Dasar

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

## 4. Konfigurasi SSH (Port Custom 2222)

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

## 5. Konfigurasi Samba (File Sharing)

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

## 6. Konfigurasi DNS Server (BIND9) untuk `[DOMAIN]` dan `webmail.[DOMAIN]`

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
sudo cp /etc/bind/db.local /etc/bind/db.[DOMAIN]
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
webmail IN      A       [IP_PUBLIK]
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

## 7. Konfigurasi Virtual Host (Apache)

### Buat direktori:
```bash
sudo mkdir -p /var/www/[DOMAIN]/public_html
sudo mkdir -p /var/www/webmail.[DOMAIN]/public_html
sudo chown -R $USER:$USER /var/www/[DOMAIN]/public_html
sudo chown -R $USER:$USER /var/www/webmail.[DOMAIN]/public_html
sudo chmod -R 755 /var/www/[DOMAIN]
sudo chmod -R 755 /var/www/webmail.[DOMAIN]
```

### Buat file index.html untuk testing:
```bash
echo "<h1>Selamat datang di [DOMAIN]</h1>" | sudo tee /var/www/[DOMAIN]/public_html/index.html
echo "<h1>Selamat datang di webmail.[DOMAIN]</h1>" | sudo tee /var/www/webmail.[DOMAIN]/public_html/index.html
```

### Buat virtual host untuk domain utama:
```bash
sudo nano /etc/apache2/sites-available/[DOMAIN].conf
```

### Isi dengan:
```apache
<VirtualHost *:80>
    ServerName [DOMAIN]
    ServerAlias www.[DOMAIN]
    DocumentRoot /var/www/[DOMAIN]/public_html

    <Directory /var/www/[DOMAIN]/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Buat virtual host untuk webmail:
```bash
sudo nano /etc/apache2/sites-available/webmail.[DOMAIN].conf
```

### Isi dengan:
```apache
<VirtualHost *:80>
    ServerName webmail.[DOMAIN]
    DocumentRoot /var/www/webmail.[DOMAIN]/public_html

    <Directory /var/www/webmail.[DOMAIN]/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Aktifkan sites dan modules:
```bash
sudo a2ensite [DOMAIN].conf
sudo a2ensite webmail.[DOMAIN].conf
sudo a2dissite 000-default.conf
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### Konfigurasi UFW untuk Apache:
```bash
sudo ufw allow from 192.168.56.0/24 to any port 80 proto tcp && sudo ufw reload
```

### Verifikasi: Akses `http://[DOMAIN]` dan `http://webmail.[DOMAIN]`

---

## 8. Konfigurasi SSL Self-Signed (Testing Lokal)

### Generate certificate untuk domain utama:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/[DOMAIN].key \
    -out /etc/ssl/certs/[DOMAIN].crt \
    -subj "/C=ID/ST=Jakarta/L=Jakarta/O=Company/OU=IT/CN=[DOMAIN]"
```

### Generate certificate untuk webmail:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/webmail.[DOMAIN].key \
    -out /etc/ssl/certs/webmail.[DOMAIN].crt \
    -subj "/C=ID/ST=Jakarta/L=Jakarta/O=Company/OU=IT/CN=webmail.[DOMAIN]"
```

### Enable SSL module:
```bash
sudo a2enmod ssl
sudo systemctl restart apache2
```

### Edit virtual host untuk SSL (domain utama):
```bash
sudo nano /etc/apache2/sites-available/[DOMAIN].conf
```

### Tambahkan di bawah konfigurasi existing:
```apache
<VirtualHost *:443>
    ServerName [DOMAIN]
    ServerAlias www.[DOMAIN]
    DocumentRoot /var/www/[DOMAIN]/public_html

    <Directory /var/www/[DOMAIN]/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/[DOMAIN].crt
    SSLCertificateKeyFile /etc/ssl/private/[DOMAIN].key
</VirtualHost>
```

### Edit virtual host untuk SSL (webmail):
```bash
sudo nano /etc/apache2/sites-available/webmail.[DOMAIN].conf
```

### Tambahkan di bawah konfigurasi existing:
```apache
<VirtualHost *:443>
    ServerName webmail.[DOMAIN]
    DocumentRoot /var/www/webmail.[DOMAIN]/public_html

    <Directory /var/www/webmail.[DOMAIN]/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/webmail.[DOMAIN].crt
    SSLCertificateKeyFile /etc/ssl/private/webmail.[DOMAIN].key
</VirtualHost>
```

### Restart Apache:
```bash
sudo systemctl restart apache2
```

### Konfigurasi UFW untuk HTTPS:
```bash
sudo ufw allow from 192.168.56.0/24 to any port 443 proto tcp && sudo ufw reload
```

### Verifikasi: Akses `https://[DOMAIN]` dan `https://webmail.[DOMAIN]`

---

## 9. Verifikasi Akhir dan Testing

### Test semua service:
```bash
# Test SSH
ssh -p 2222 [NAMA_USER]@[IP_HOST_ONLY]

# Test DNS
dig @localhost [DOMAIN]
dig @localhost webmail.[DOMAIN]

# Test Apache
curl -I http://[DOMAIN]
curl -I https://[DOMAIN]

# Test Samba
smbclient -L //[IP_HOST_ONLY]/shared -N
```

### Check status services:
```bash
sudo systemctl status ssh
sudo systemctl status bind9
sudo systemctl status apache2
sudo systemctl status smbd
sudo ufw status
```

### Test dari Windows:
1. **SSH:** PuTTY ke `[IP_HOST_ONLY]:2222`
2. **Web:** Browser ke `http://[DOMAIN]` dan `https://[DOMAIN]`
3. **File Sharing:** `\\[IP_HOST_ONLY]\shared`
4. **DNS:** Ping `[DOMAIN]` dan `webmail.[DOMAIN]`

---

## 10. Troubleshooting

### SSH tidak bisa connect:
- Cek UFW: `sudo ufw status`
- Cek service SSH: `sudo systemctl status ssh`
- Cek port: `sudo ss -tlnp | grep :2222`

### DNS tidak resolve:
- Cek BIND9: `sudo systemctl status bind9`
- Test konfigurasi: `sudo named-checkzone [DOMAIN] /etc/bind/db.[DOMAIN]`
- Cek log: `sudo journalctl -u bind9`

### Apache tidak accessible:
- Cek Apache: `sudo systemctl status apache2`
- Test konfigurasi: `sudo apache2ctl configtest`
- Cek log: `sudo tail -f /var/log/apache2/error.log`

### Samba tidak accessible:
- Cek Samba: `sudo systemctl status smbd`
- Test konfigurasi: `sudo testparm`
- Cek log: `sudo tail -f /var/log/samba/log.smbd`

---

## Selesai!

Server Ubuntu 24.04 LTS Anda telah dikonfigurasi dengan:
- ✅ User dengan hak sudo
- ✅ Jaringan ganda (NAT + Host-Only)
- ✅ SSH dengan port custom 2222
- ✅ DNS server BIND9
- ✅ Web server Apache dengan virtual host
- ✅ SSL self-signed certificate
- ✅ File sharing Samba
- ✅ Firewall UFW terkonfigurasi

Server siap digunakan untuk pengembangan dan testing internal.
