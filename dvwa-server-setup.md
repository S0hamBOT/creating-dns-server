# DVWA Pentesting Lab — Complete Setup Guide
> **Author:** Soham Jadhav | **Domain:** `dvwa.sohamjadhav.in` | **Platform:** VMware Workstation on Windows

---

## Lab Architecture

```
Host Machine (Windows)
│
├── VMware Workstation
│   ├── DNS Server VM   →  192.168.1.10   (BIND9)
│   └── DVWA Server VM  →  192.168.1.20   (Apache + PHP + MySQL + DVWA)
│
└── Network
    ├── ens33 → Host-only (VMnet1) → 192.168.1.0/24  ← lab communication
    └── ens37 → NAT       (VMnet8)                   ← internet access
```

### IP Plan

| Machine | IP Address |
|---|---|
| DNS Server | 192.168.1.10 |
| DVWA Server | 192.168.1.20 |
| Kali / Scanner | 192.168.1.X |

---

## Part 1 — DNS Server VM (192.168.1.10)

### 1.1 Install BIND9

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y bind9 bind9utils bind9-doc dnsutils net-tools
```

### 1.2 Configure named.conf.options

```bash
sudo nano /etc/bind/named.conf.options
```

```
options {
    directory "/var/cache/bind";

    allow-query {
        localhost;
        192.168.1.0/24;
    };

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    forward only;
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { none; };
    recursion yes;
    allow-recursion {
        localhost;
        192.168.1.0/24;
    };
};
```

### 1.3 Configure named.conf.local

```bash
sudo nano /etc/bind/named.conf.local
```

```
zone "sohamjadhav.in" {
    type master;
    file "/etc/bind/zones/db.sohamjadhav.in";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

### 1.4 Create Zone Files

```bash
sudo mkdir -p /etc/bind/zones
```

**Forward zone** — `/etc/bind/zones/db.sohamjadhav.in`:

```bash
sudo nano /etc/bind/zones/db.sohamjadhav.in
```

```dns
$TTL    604800
@       IN      SOA     ns1.sohamjadhav.in. admin.sohamjadhav.in. (
                        2024010101      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        604800 )        ; Negative Cache TTL

@       IN      NS      ns1.sohamjadhav.in.
ns1     IN      A       192.168.1.10

; ── Lab Apps ──────────────────────────────────
dvwa    IN      A       192.168.1.20
; juice  IN      A       192.168.1.21   ← add future apps here
; ──────────────────────────────────────────────
```

**Reverse zone** — `/etc/bind/zones/db.192.168.1`:

```bash
sudo nano /etc/bind/zones/db.192.168.1
```

```dns
$TTL    604800
@       IN      SOA     ns1.sohamjadhav.in. admin.sohamjadhav.in. (
                        2024010101
                        3600
                        1800
                        604800
                        604800 )

@       IN      NS      ns1.sohamjadhav.in.
10      IN      PTR     ns1.sohamjadhav.in.
20      IN      PTR     dvwa.sohamjadhav.in.
```

### 1.5 Validate and Start BIND9

```bash
sudo named-checkconf
sudo named-checkzone sohamjadhav.in /etc/bind/zones/db.sohamjadhav.in
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1

sudo systemctl restart named
sudo systemctl enable named
```

### 1.6 Test DNS Resolution

```bash
dig @127.0.0.1 dvwa.sohamjadhav.in
nslookup dvwa.sohamjadhav.in 127.0.0.1
```

Expected output:
```
Name:   dvwa.sohamjadhav.in
Address: 192.168.1.20
```

### 1.7 Netplan — DNS Server Network Config

File: `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.1.10/24
      nameservers:
        addresses:
          - 127.0.0.1

    ens37:
      dhcp4: yes
```

```bash
sudo netplan apply
```

> **Important notes:**
> - Do NOT set a gateway on `ens33` (host-only)
> - Default route must come from `ens37` (NAT) via DHCP
> - DNS points to `127.0.0.1` (BIND9 running locally)

---

## Part 2 — DVWA Server VM (192.168.1.20)

### 2.1 Clone the DNS Server VM

1. Shut down the DNS Server VM:
   ```bash
   sudo shutdown -h now
   ```
2. In VMware: right-click DNS VM → **Manage** → **Clone**
3. Clone type: **Full Clone** | Name: `DVWA-Server`
4. Power on the clone

### 2.2 Post-Clone Configuration

**Change hostname:**
```bash
sudo hostnamectl set-hostname dvwa-server
exec bash
```

**Fix sudo warning (hostname not in /etc/hosts):**
```bash
echo "127.0.0.1 dvwa-server" | sudo tee -a /etc/hosts
```

**Set static IP:**

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.1.20/24
      nameservers:
        addresses:
          - 192.168.1.10

    ens37:
      dhcp4: yes
```

```bash
sudo netplan apply
ip -4 addr show ens33   # verify: inet 192.168.1.20/24
```

**Remove BIND9 (not needed on DVWA VM):**
```bash
sudo systemctl stop named
sudo systemctl disable named
sudo apt remove --purge bind9 bind9utils -y
sudo apt autoremove -y
```

**Disable cloud-init network management (prevents config being overwritten on reboot):**
```bash
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network.cfg'
```

**Verify DNS works from DVWA VM:**
```bash
nslookup dvwa.sohamjadhav.in 192.168.1.10
# Expected: Address: 192.168.1.20
```

**Take a snapshot before installing anything:**
> VMware → Right-click `DVWA-Server` → Snapshot → Take Snapshot → `"Fresh Clone - No DVWA"`

---

### 2.3 Install Apache

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apache2
sudo systemctl start apache2
sudo systemctl enable apache2
```

Verify:
```bash
curl http://localhost   # returns Apache default HTML
```

---

### 2.4 Install PHP + Extensions

```bash
sudo apt install -y php php-mysqli php-gd php-xml php-mbstring \
php-tokenizer php-curl libapache2-mod-php
```

Verify:
```bash
php -v   # Should show PHP 8.3.x
```

Enable PHP module:
```bash
sudo a2enmod php8.3
sudo systemctl restart apache2
```

**Update PHP settings:**
```bash
sudo nano /etc/php/8.3/apache2/php.ini
```

Change these values:
```ini
display_errors = On
display_startup_errors = On
allow_url_include = On
allow_url_fopen = On
```

---

### 2.5 Install MySQL

```bash
sudo apt install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
```

**Secure MySQL:**
```bash
sudo mysql_secure_installation
```

Recommended answers:
```
Validate password component?   → N
New root password:             → dvwaroot
Remove anonymous users?        → Y
Disallow root login remotely?  → Y
Remove test database?          → Y
Reload privilege tables?       → Y
```

**Create DVWA database and user:**
```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE dvwa;

-- Create user for both localhost and 127.0.0.1 (DVWA config uses 127.0.0.1)
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'p@ssw0rd';
CREATE USER 'dvwa'@'127.0.0.1' IDENTIFIED BY 'p@ssw0rd';

GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'127.0.0.1';
FLUSH PRIVILEGES;
EXIT;
```

> **Why both `localhost` AND `127.0.0.1`?**
> MySQL treats `localhost` (socket) and `127.0.0.1` (TCP) as different hosts.
> DVWA's config connects via `127.0.0.1`, so both entries are required.

---

### 2.6 Clone and Configure DVWA

```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git dvwa
sudo chown -R www-data:www-data /var/www/html/dvwa
sudo chmod -R 755 /var/www/html/dvwa
```

**Create config file:**
```bash
sudo cp /var/www/html/dvwa/config/config.inc.php.dist \
        /var/www/html/dvwa/config/config.inc.php

sudo nano /var/www/html/dvwa/config/config.inc.php
```

Set these values:
```php
$_DVWA[ 'db_server' ]              = '127.0.0.1';
$_DVWA[ 'db_database' ]            = 'dvwa';
$_DVWA[ 'db_user' ]                = 'dvwa';
$_DVWA[ 'db_password' ]            = 'p@ssw0rd';
$_DVWA[ 'db_port']                 = '3306';
$_DVWA[ 'default_security_level' ] = 'low';
```

**Set file permissions:**
```bash
sudo chmod 777 /var/www/html/dvwa/hackable/uploads/
sudo chmod 777 /var/www/html/dvwa/config/

# Create phpids log manually if it doesn't exist
sudo mkdir -p /var/www/html/dvwa/external/phpids/0.6/lib/IDS/tmp
sudo touch /var/www/html/dvwa/external/phpids/0.6/lib/IDS/tmp/phpids_log.txt
sudo chmod 666 /var/www/html/dvwa/external/phpids/0.6/lib/IDS/tmp/phpids_log.txt
```

---

### 2.7 Configure Apache VirtualHost

```bash
sudo nano /etc/apache2/sites-available/dvwa.conf
```

```apache
<VirtualHost *:80>
    ServerName dvwa.sohamjadhav.in
    ServerAlias www.dvwa.sohamjadhav.in
    DocumentRoot /var/www/html/dvwa

    <Directory /var/www/html/dvwa>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/dvwa_error.log
    CustomLog ${APACHE_LOG_DIR}/dvwa_access.log combined
</VirtualHost>
```

```bash
sudo a2ensite dvwa.conf
sudo a2dissite 000-default.conf
sudo a2enmod rewrite
```

**Fix Apache ServerName warning:**
```bash
echo "ServerName dvwa.sohamjadhav.in" | sudo tee -a /etc/apache2/apache2.conf
```

```bash
sudo systemctl restart apache2
```

**Verify VirtualHost is active:**
```bash
sudo apache2ctl -S
# Should show: dvwa.sohamjadhav.in (/etc/apache2/sites-enabled/dvwa.conf:1)
```

---

### 2.8 Fix DVWA SQL Compatibility Bug (Critical)

> **What happened:** DVWA's `MySQL.php` uses `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`
> which causes a syntax error through DVWA's custom `mysqli` wrapper (`___mysqli_ston`),
> even though MySQL 8.0 technically supports it. The wrapper mangles the query
> before MySQL parses it, causing HTTP 500 on database setup.
>
> **Fix:** Remove the `IF NOT EXISTS` condition. Safe to do because the database
> is always dropped and recreated fresh during setup — columns never pre-exist.

**Apply the patch:**
```bash
sudo sed -i 's/ADD COLUMN IF NOT EXISTS/ADD COLUMN/g' \
/var/www/html/dvwa/dvwa/includes/DBMS/MySQL.php
```

**Verify the fix:**
```bash
grep "ALTER TABLE" /var/www/html/dvwa/dvwa/includes/DBMS/MySQL.php
# Must NOT contain "IF NOT EXISTS"
```

---

### 2.9 Initialize DVWA Database

**Drop and recreate clean database:**
```bash
sudo mysql -u root -p
```

```sql
DROP DATABASE IF EXISTS dvwa;
CREATE DATABASE dvwa;
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'127.0.0.1';
FLUSH PRIVILEGES;
EXIT;
```

**Restart Apache:**
```bash
sudo systemctl restart apache2
```

**Verify PHP can connect to MySQL:**
```bash
php -r "
\$conn = new mysqli('127.0.0.1', 'dvwa', 'p@ssw0rd', 'dvwa');
if (\$conn->connect_error) {
    echo 'FAILED: ' . \$conn->connect_error . PHP_EOL;
} else {
    echo 'SUCCESS: PHP connected to MySQL' . PHP_EOL;
}
"
# Must output: SUCCESS: PHP connected to MySQL
```

**Open in browser and click "Create / Reset Database":**
```
http://192.168.1.20/setup.php
    OR
http://dvwa.sohamjadhav.in/setup.php   ← if DNS is configured on your host
```

Click **"Create / Reset Database"** → page redirects to login automatically.

**Verify tables were created:**
```bash
mysql -u dvwa -p'p@ssw0rd' -h 127.0.0.1 dvwa -e "SHOW TABLES;"
```

Expected:
```
+----------------+
| Tables_in_dvwa |
+----------------+
| guestbook      |
| users          |
+----------------+
```

---

### 2.10 Login to DVWA

```
URL:       http://dvwa.sohamjadhav.in/login.php
Username:  admin
Password:  password
```

---

## Part 3 — Configure Kali / Scanner VM

Set DNS to point to your BIND9 server:

```bash
sudo nano /etc/resolv.conf
```

```
nameserver 192.168.1.10
```

Make it permanent via netplan:
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Add under your interface:
```yaml
nameservers:
  addresses:
    - 192.168.1.10
```

```bash
sudo netplan apply
```

**Test from Kali:**
```bash
nslookup dvwa.sohamjadhav.in     # Should return 192.168.1.20
curl http://dvwa.sohamjadhav.in  # Should return DVWA HTML
```

---

## Part 4 — Adding Future Lab Apps

When you spin up a new VM (e.g. Juice Shop at `192.168.1.21`):

**Step 1 — Add to DNS forward zone** (`/etc/bind/zones/db.sohamjadhav.in`):
```dns
juice   IN      A       192.168.1.21
```

**Step 2 — Add to reverse zone** (`/etc/bind/zones/db.192.168.1`):
```dns
21      IN      PTR     juice.sohamjadhav.in.
```

**Step 3 — Increment serial in both zone files, then reload:**
```bash
# Change serial e.g. 2024010101 → 2024010102
sudo named-checkconf && sudo systemctl reload named
```

---

## Troubleshooting Reference

### DVWA Setup HTTP 500
```bash
sudo tail -20 /var/log/apache2/dvwa_error.log
```
Most common cause: SQL syntax error in `MySQL.php` → apply the `IF NOT EXISTS` patch (Section 2.8).

### DNS Not Resolving
```bash
dig @192.168.1.10 dvwa.sohamjadhav.in
sudo systemctl status named
sudo named-checkconf
```

### Apache Not Starting
```bash
sudo apache2ctl configtest
sudo journalctl -u apache2 --no-pager | tail -20
```

### MySQL Connection Refused
```bash
sudo systemctl status mysql
mysql -u dvwa -p'p@ssw0rd' -h 127.0.0.1 dvwa
```

### No Internet on DVWA VM (ens37)
```bash
ip route show                    # check for default route
sudo networkctl status ens37     # check DHCP state
sudo networkctl up ens37
sudo networkctl renew ens37

# If still no IP — assign manually (replace subnet with your VMnet8):
sudo ip addr add 192.168.163.200/24 dev ens37
sudo ip route add default via 192.168.163.2 dev ens37
```

### sudo Warning: "unable to resolve host dvwa-server"
```bash
echo "127.0.0.1 dvwa-server" | sudo tee -a /etc/hosts
```

---

## Final State Verification Checklist

```
[ ] DNS Server running:       sudo systemctl status named
[ ] dvwa.sohamjadhav.in resolves to 192.168.1.20
[ ] Apache running:           sudo systemctl status apache2
[ ] MySQL running:            sudo systemctl status mysql
[ ] PHP → MySQL connected:    php -r "..." (Section 2.9)
[ ] MySQL.php patched:        grep "ALTER TABLE" MySQL.php (no IF NOT EXISTS)
[ ] DVWA tables exist:        SHOW TABLES; returns guestbook + users
[ ] Login works:              http://dvwa.sohamjadhav.in/login.php
[ ] Kali DNS set to:          192.168.1.10
```

---

*Isolated lab — `sohamjadhav.in` resolves only inside the `192.168.1.0/24` host-only network and does not touch Cloudflare or public DNS.*
