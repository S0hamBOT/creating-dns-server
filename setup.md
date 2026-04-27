## VMware Network Setup

- **Adapter 1 (ens33)** → Host-only (VMnet1)  
    → Used for lab communication  
    → Subnet: `192.168.100.0/24`
    
- **Adapter 2 (ens37)** → NAT (VMnet8)  
    → Used for internet access
    

---

## Netplan Configuration File

Path:

```bash
/etc/netplan/01-netcfg.yaml
```

---

## Final Configuration

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    ens33:  # Host-only (Lab Network)
      dhcp4: no
      addresses:
        - 192.168.100.10/24
      nameservers:
        addresses:
          - 127.0.0.1   # Local BIND9 DNS

    ens37:  # NAT (Internet Access)
      dhcp4: yes
```

---

## Apply Configuration

```bash
sudo netplan apply
```

---

## Verify Network

```bash
ip a
ip route
```

### Expected:

- `ens33 → 192.168.100.10`
    
- `ens37 → 192.168.xxx.xxx` (from NAT)
    

---

## Test Connectivity

```bash
# Internet (should work via NAT)
ping 8.8.8.8

# DNS resolution via BIND9
ping google.com
```

---

## Important Notes

- ❌ **Do NOT set gateway on ens33 (host-only)**
    
- ✅ Default route must come from **ens37 (NAT)**
    
- ✅ DNS points to `127.0.0.1` (your BIND server)
    
- ✅ BIND should forward external queries to:
    
    ```
    8.8.8.8
    8.8.4.4
    ```
    

---

## Lab IP Plan

|Machine|IP Address|
|---|---|
|DNS Server|192.168.100.10|
|DVWA Server|192.168.100.20|
|Kali Linux|192.168.100.X|

---

## Final Architecture

- Host-only → isolated pentesting lab
    
- NAT → safe internet access
    
- DNS server → resolves:
    
    - Internal domains (`dvwa.sohamjadhav.in`)
        
    - External domains (`google.com`)
        

---

# Setting Up a Local DNS Server for Your Pentesting Lab

Here's how to set up a **BIND9 DNS server** on a dedicated VM that you can reuse for any future lab application.

---

## Phase 1: DNS Server VM Setup

### 1.1 — Install BIND9

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y bind9 bind9utils bind9-doc dnsutils net-tools
```

### 1.2 — Get Your VM's IP (note this down)

```bash
ip -4 addr show | grep inet
# Example output: inet 192.168.100.10/24
# Your DNS Server IP = 192.168.100.10
```

---

## Phase 2: Configure BIND9

### 2.1 — Edit the main named.conf.options

```bash
sudo nano /etc/bind/named.conf.options
```

Paste this entire block:

```bash
options {
    directory "/var/cache/bind";

    # Allow queries from your lab network only
    allow-query { 
        localhost; 
        192.168.100.0/24;   # Change to match YOUR subnet
    };

    # Forward unknown queries to upstream DNS (Google)
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    forward only;

    # Security hardening
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { none; };
    recursion yes;
    allow-recursion { 
        localhost; 
        192.168.100.0/24;   # Same subnet as above
    };
};
```

### 2.2 — Declare your zone in named.conf.local

```bash
sudo nano /etc/bind/named.conf.local
```

```bash
# Forward Zone — resolves hostnames to IPs
zone "sohamjadhav.in" {
    type master;
    file "/etc/bind/zones/db.sohamjadhav.in";
};

# Reverse Zone — resolves IPs back to hostnames
zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.100";
};
```

> ⚠️ The reverse zone name is your subnet **written backwards**. For `192.168.100.x` it becomes `100.168.192.in-addr.arpa`

---

## Phase 3: Create Zone Files

### 3.1 — Create the zones directory

```bash
sudo mkdir -p /etc/bind/zones
```

### 3.2 — Forward zone file

```bash
sudo nano /etc/bind/zones/db.sohamjadhav.in
```

```dns
$TTL    604800
@       IN      SOA     ns1.sohamjadhav.in. admin.sohamjadhav.in. (
                        2024010101      ; Serial  <-- increment this when you make changes
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        604800 )        ; Negative Cache TTL

; Name Servers
@       IN      NS      ns1.sohamjadhav.in.

; A Records — NS itself
ns1     IN      A       192.168.100.10   ; DNS Server's own IP

; ── Add your lab apps below this line ──────────────────
dvwa    IN      A       192.168.100.20   ; DVWA VM IP (to be set up later)
; juice   IN     A       192.168.100.21   ; Example: future app
; bwapp   IN     A       192.168.100.22   ; Example: future app
; ────────────────────────────────────────────────────────
```

### 3.3 — Reverse zone file

```bash
sudo nano /etc/bind/zones/db.192.168.100
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

; PTR Records — last octet only
10      IN      PTR     ns1.sohamjadhav.in.    ; DNS server itself
20      IN      PTR     dvwa.sohamjadhav.in.   ; DVWA VM
```

---

## Phase 4: Validate and Start BIND9

### 4.1 — Check config syntax (must show no errors)

```bash
sudo named-checkconf
sudo named-checkzone sohamjadhav.in /etc/bind/zones/db.sohamjadhav.in
sudo named-checkzone 100.168.192.in-addr.arpa /etc/bind/zones/db.192.168.100
```

Expected output for zone checks:

```
zone sohamjadhav.in/IN: loaded serial 2024010101
OK
```

### 4.2 — Start and enable BIND9

```bash
sudo systemctl restart bind9
sudo systemctl enable bind9
sudo systemctl status bind9
```

### 4.3 — Open firewall port for DNS

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw reload
sudo ufw status
```

---

## Phase 5: Test the DNS Server

Run these from the **DNS server itself** first:

```bash
# Test forward resolution
dig @127.0.0.1 dvwa.sohamjadhav.in

# Test reverse resolution
dig @127.0.0.1 -x 192.168.100.20

# Quick nslookup test
nslookup dvwa.sohamjadhav.in 127.0.0.1
```

Expected output from `dig`:

```
;; ANSWER SECTION:
dvwa.sohamjadhav.in.  604800  IN  A  192.168.100.20
```

---

## Phase 6: How to Add Any New App Later

When you spin up a new VM (e.g., Juice Shop at `192.168.100.21`), you only need to do **2 things**:

**Step 1** — Add to forward zone (`db.sohamjadhav.in`):

```dns
juice   IN      A       192.168.100.21
```

**Step 2** — Add to reverse zone (`db.192.168.100`):

```dns
21      IN      PTR     juice.sohamjadhav.in.
```

**Step 3** — Bump the serial and reload:

```bash
# Increment the serial number in BOTH zone files first, then:
sudo named-checkconf && sudo systemctl reload bind9
```

---

## Network Topology So Far

```
┌─────────────────────────────────────────────┐
│           Isolated Host-Only Network         │
│              192.168.100.0/24                │
│                                              │
│  ┌──────────────┐      ┌──────────────────┐  │
│  │  DNS Server  │      │   DVWA VM        │  │
│  │ 192.168.100.10│      │ 192.168.100.20   │  │
│  │  (BIND9)     │      │ (to be set up)   │  │
│  └──────────────┘      └──────────────────┘  │
│                                              │
│  ┌──────────────┐                            │
│  │ Scanner/Kali │                            │
│  │ 192.168.100.X│                            │
│  └──────────────┘                            │
└─────────────────────────────────────────────┘
```

---

