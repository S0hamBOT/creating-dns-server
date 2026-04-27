# Pentesting Lab — Network & DNS Server Setup Guide
> **Platform:** VMware Workstation | **Domain:** `sohamjadhav.in` | **Subnet:** `192.168.1.0/24`

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│              Isolated Host-Only Network              │
│                  192.168.1.0/24                    │
│                                                      │
│  ┌───────────────┐        ┌───────────────────────┐  │
│  │  DNS Server   │        │       DVWA VM         │  │
│  │192.168.1.10 │        │   192.168.1.20      │  │
│  │   (BIND9)     │        │   (to be set up)      │  │
│  └───────────────┘        └───────────────────────┘  │
│                                                      │
│  ┌───────────────┐                                   │
│  │ Scanner/Kali  │                                   │
│  │192.168.1.X  │                                   │
│  └───────────────┘                                   │
└─────────────────────────────────────────────────────┘
         │
         │ ens37 (NAT / VMnet8)
         ▼
    Internet (8.8.8.8)
```

### IP Plan

| Machine | IP Address |
|---|---|
| DNS Server | 192.168.1.10 |
| DVWA Server | 192.168.1.20 |
| Kali / Scanner | 192.168.1.X |

---

## Part 1 — VMware Network Adapter Setup

Configure **two network adapters** on the DNS Server VM in VMware settings:

| Adapter | VMware Network | Purpose | Subnet |
|---|---|---|---|
| Adapter 1 (`ens33`) | Host-only (VMnet1) | Lab communication | `192.168.1.0/24` |
| Adapter 2 (`ens37`) | NAT (VMnet8) | Internet access | DHCP from VMware |

> **How to set this in VMware:**
> VM Settings → Add → Network Adapter → select Host-only or NAT accordingly

---

## Part 2 — DNS Server Network Configuration (Netplan)

### 2.1 Configuration File

Path: `/etc/netplan/50-cloud-init.yaml`

> **Note:** On Ubuntu 24.04 the active file may be `50-cloud-init.yaml` instead of `01-netcfg.yaml`.
> Check with `ls /etc/netplan/` and edit whichever file exists.

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    ens33:  # Host-only — Lab Network
      dhcp4: no
      addresses:
        - 192.168.1.10/24
      nameservers:
        addresses:
          - 127.0.0.1   # Local BIND9 DNS

    ens37:  # NAT — Internet Access
      dhcp4: yes
```

### 2.2 Apply and Verify

```bash
sudo netplan apply
```

```bash
ip a        # Check assigned IPs
ip route    # Check routing table
```

Expected results:
- `ens33` → `192.168.1.10`
- `ens37` → `192.168.x.x` (assigned by VMware DHCP)
- Default route via `ens37`

### 2.3 Test Connectivity

```bash
# Internet via NAT
ping -c 3 8.8.8.8

# DNS resolution via BIND9
ping -c 3 google.com
```

### 2.4 Important Rules

| Rule | Detail |
|---|---|
| ❌ No gateway on `ens33` | Host-only must NOT have a default route |
| ✅ Default route via `ens37` | NAT adapter provides internet via DHCP |
| ✅ DNS → `127.0.0.1` | BIND9 runs locally on this VM |
| ✅ Forward external queries | BIND9 forwards to `8.8.8.8` / `8.8.4.4` |

---

## Part 3 — BIND9 DNS Server Installation

### 3.1 Install BIND9

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y bind9 bind9utils bind9-doc dnsutils net-tools
```

### 3.2 Confirm Your VM's IP

```bash
ip -4 addr show | grep inet
# Look for ens33 — should show 192.168.1.10/24
```

---

## Part 4 — Configure BIND9

### 4.1 Edit named.conf.options

```bash
sudo nano /etc/bind/named.conf.options
```

```
options {
    directory "/var/cache/bind";

    # Allow queries from lab network only
    allow-query {
        localhost;
        192.168.1.0/24;
    };

    # Forward unknown queries to Google DNS
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    forward only;

    # Security
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

### 4.2 Declare Zones in named.conf.local

```bash
sudo nano /etc/bind/named.conf.local
```

```
# Forward Zone — resolves hostnames to IPs
zone "sohamjadhav.in" {
    type master;
    file "/etc/bind/zones/db.sohamjadhav.in";
};

# Reverse Zone — resolves IPs back to hostnames
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

> **How reverse zone names work:**
> Subnet `192.168.1.x` written backwards → `1.168.192.in-addr.arpa`

---

## Part 5 — Create Zone Files

### 5.1 Create Zones Directory

```bash
sudo mkdir -p /etc/bind/zones
```

### 5.2 Forward Zone File

```bash
sudo nano /etc/bind/zones/db.sohamjadhav.in
```

```dns
$TTL    604800
@       IN      SOA     ns1.sohamjadhav.in. admin.sohamjadhav.in. (
                        2024010101      ; Serial — increment on every change
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        604800 )        ; Negative Cache TTL

; Name Servers
@       IN      NS      ns1.sohamjadhav.in.

; DNS Server itself
ns1     IN      A       192.168.1.10

; ── Lab Apps — add new entries below ──────────────────
dvwa    IN      A       192.168.1.20
; juice  IN      A       192.168.1.21   ← future app example
; bwapp  IN      A       192.168.1.22   ← future app example
; ──────────────────────────────────────────────────────
```

### 5.3 Reverse Zone File

```bash
sudo nano /etc/bind/zones/db.192.168.1
```

```dns
$TTL    604800
@       IN      SOA     ns1.sohamjadhav.in. admin.sohamjadhav.in. (
                        2024010101      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        604800 )        ; Negative Cache TTL

; Name Servers
@       IN      NS      ns1.sohamjadhav.in.

; PTR Records — last octet of IP only
10      IN      PTR     ns1.sohamjadhav.in.
20      IN      PTR     dvwa.sohamjadhav.in.
```

---

## Part 6 — Validate and Start BIND9

### 6.1 Check Config Syntax

```bash
sudo named-checkconf
sudo named-checkzone sohamjadhav.in /etc/bind/zones/db.sohamjadhav.in
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1
```

Expected output:
```
zone sohamjadhav.in/IN: loaded serial 2024010101
OK
```

Both zone checks must return `OK` before proceeding.

### 6.2 Start and Enable BIND9

```bash
sudo systemctl restart named
sudo systemctl enable named
sudo systemctl status named
```

> **Note:** On Ubuntu 24.04, the service is called `named` not `bind9`.
> `bind9` is an alias — use `named` to avoid the "refusing to operate on alias" warning.

### 6.3 Open Firewall for DNS

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw reload
sudo ufw status
```

---

## Part 7 — Test DNS Resolution

Run all tests from the DNS server itself first:

```bash
# Forward resolution — hostname to IP
dig @127.0.0.1 dvwa.sohamjadhav.in

# Reverse resolution — IP to hostname
dig @127.0.0.1 -x 192.168.1.20

# Quick test
nslookup dvwa.sohamjadhav.in 127.0.0.1
```

Expected `dig` output:
```
;; ANSWER SECTION:
dvwa.sohamjadhav.in.  604800  IN  A  192.168.1.20
```

---

## Part 8 — Adding Future Lab Apps

When you spin up any new VM, only 3 steps needed on the DNS server:

**Step 1 — Forward zone** (`/etc/bind/zones/db.sohamjadhav.in`):
```dns
juice   IN      A       192.168.1.21
```

**Step 2 — Reverse zone** (`/etc/bind/zones/db.192.168.1`):
```dns
21      IN      PTR     juice.sohamjadhav.in.
```

**Step 3 — Increment serial in both files, then reload:**
```bash
# Edit both zone files and bump serial e.g. 2024010101 → 2024010102
sudo named-checkconf && sudo systemctl reload named
```

---

## Troubleshooting Reference

### BIND9 won't start
```bash
sudo named-checkconf                  # check config syntax
sudo journalctl -u named --no-pager   # check service logs
```

### DNS not resolving
```bash
dig @127.0.0.1 dvwa.sohamjadhav.in   # test locally first
sudo named-checkzone sohamjadhav.in /etc/bind/zones/db.sohamjadhav.in
```

### No internet on VM (ens37 has no IP)
```bash
ip route show                   # check for default route
sudo networkctl status ens37    # check DHCP state
sudo networkctl up ens37
sudo networkctl renew ens37
```

### sudo warning: "unable to resolve host"
```bash
echo "127.0.0.1 $(hostname)" | sudo tee -a /etc/hosts
```

### Cloud-init overwriting netplan on reboot
```bash
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network.cfg'
```

---

## Final Verification Checklist

```
[ ] ens33 has IP 192.168.1.10
[ ] ens37 has IP from DHCP (NAT)
[ ] Default route is via ens37
[ ] ping 8.8.8.8 works
[ ] named service is active and enabled
[ ] named-checkconf returns no errors
[ ] dig @127.0.0.1 dvwa.sohamjadhav.in returns 192.168.1.20
[ ] Port 53 open (ufw or firewall inactive)
```

---

*This DNS server is reusable — spin up any new lab VM, add one DNS record, reload BIND9, and the domain resolves instantly across your entire lab network.*
