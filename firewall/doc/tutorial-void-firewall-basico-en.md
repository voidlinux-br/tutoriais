# 🧩 VOID LINUX TUTORIAL — SECURITY SCHEME IMPLEMENTATION – VOIDBR LABORATORY

📌 Firewall Void Linux com IPTables + Unbound DNS + Port Knocking.

---

## 🔥 FIREWALL + PROXY INTEGRATED TO THE SAMBA4 DOMAIN (VOIDBR.NET)

## 🎯 OBJECTIVE in this tutorial is to configure a **Firewall** server on **Void Server**, acting as **main gateway (192.168.70.254)**.

---

## 🌐 1. Network topology - Function, IP addressing and names:

- Domain: VOIDBR.NET

- Firewall/Proxy: SRVFIREWALL 192.168.70.254

- Domain Controller: SRVDC01 192.168.70.253

- FileServer: SRVFILES 192.168.70.252

- WAN: `eth0` → 192.168.122.254/24 (gateway: 192.168.122.1)

- LAN: `eth1` → 192.168.70.254/24

---

## 📌 Deployment target

### INPUT

-> Allow ping to Firewall from LAN/DMZ
-> Allow SSH to Firewall from LAN/DMZ
-> Allow access to Proxy over LAN
-> Allow external SSH to the Firewall via the RECENT module with Port knock

### FORWARD

-> Allow LAN access AD -> DNS, SMB, DHCP, NETBIOS)
-> Allow DMZ to access the internet
-> Allow AD to access DNS on the internet
-> LAN needs to access email on the internet
-> Allow DMZ ping from LAN
-> LAN needs to access revenuenet, gov, etc.

### NAT

-> Publish Web Server -> 80/443
-> Outbound NAT for LAN and for DMZ

---

## ✅ 2. NETWORK DIAGRAM

```bash
Internet
   |
[Roteador do ISP]
LAN: 192.168.122.1/24
   |
[Firewall VM - Void Linux]
eth0 (WAN): 192.168.122.254/24
eth1 (LAN): 192.168.70.254/24
   |
[Rede interna / Switch]
```

View from another angle

```bash
Internet
  |
[ Knock correto ]
  |
iptables (xt_recent libera SSH por X segundos)
  |
sshd (porta 22254)
  |
Fail2ban (analisa auth.log)
  |
iptables (ban definitivo do IP)
```

The firewall is the only host exposed to the Internet.

## ✅ 3. OBJECTIVES AND ASSUMPTIONS

- Deny default policy
- Active IPv4 routing
- Scanner never sees the door
- Firewall as the only point of entry
- No web dashboards published
- SSH protected by Port Knocking
- Brute-force control via Fail2ban
- Controlled NAT for the LAN
- Remote administration via SSH tunnel

## Note: Tutorial running under root user!!

## ✅ 4. UPDATE AND INSTALL NECESSARY PACKAGES

Update the system

```bash
vinstall -Syu
```

Install the packages

```bash
vinstall -y \
  vim \
  bash-completion \
  iptables \
  iproute2 \
  openssh \
  tcpdump \
  conntrack-tools \
  unbound
```

## Basic configuration for the dnsmasq file

```bash
vim /etc/unbound/unbound.conf
```

Content:

```bash
server:
    interface: 127.0.0.1
    interface: 192.168.122.254
    interface: 192.168.70.254

    access-control: 127.0.0.0/8 allow
    access-control: 192.168.122.0/24 allow
    access-control: 192.168.70.0/24 allow
    access-control: 0.0.0.0/0 refuse

forward-zone:
    name: "."
    forward-addr: 1.1.1.1
    forward-addr: 8.8.8.8
```

## Enable UNBOUND

```bash
vservice enable unbound
```

## Validates online services

```bash
vsv
```

## ✅ 5. SSH CONFIGURATION

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

Add the pointed lines

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## Service activation

```bash
vservice enable sshd
```

## After full deployment:

- Disable root login

- Use only key authentication

## ✅ 6. FIREWALL NETWORK SETUP

```bash
vim /etc/dhcpcd.conf
```

Content

```bash
# CONFIGURAÇÃO DE REDE DO FIREWALL

# WAN – 192.168.122.0/24
interface eth0
static ip_address=192.168.122.254/24
static routers=192.168.122.1
static domain_name_servers=192.168.122.1 8.8.8.8

# LAN – 192.168.70.0/24
interface eth1
static ip_address=192.168.70.254/24
nogateway
```

Apply

```bash
vservice restart dhcpcd
```

## ✅ 7. PORT KNOCKING – KERNEL SUPPORT

Load the required module

```bash
modprobe xt_recent
```

Validate:

```bash
lsmod | grep xt_recent
```

Expected result

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## Add these modules persistently by creating the file

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. FIREWALL IPTABLES

Enable routing between Firewall network cards

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

Apply without reboot:

```bash
sysctl --system
```

Clear all prerules (if any):

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 FILTER table rules

Set default policies:

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

INPUT (firewall access)

```bash
# Libera loopback
iptables -A INPUT -i lo -j ACCEPT

# Conexões estabelecidas
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

# ICMP (ping) da LAN
iptables -A INPUT -s 192.168.70.0/24 -p icmp --icmp-type echo-request -j ACCEPT

iptables -A INPUT -i eth1 -p udp --dport 53 -s 192.168.70.0/24 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 53 -s 192.168.70.0/24 -j ACCEPT


# SSH (porta customizada)
iptables -A INPUT -s 192.168.70.0/24 -p tcp --dport 22254 -j ACCEPT
```

🔐 FORWARD table rules (traffic between networks)

```bash
# Conexões estabelecidas
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

# ICMP (opcional, recomendado)
iptables -A FORWARD -p icmp -j ACCEPT

# Libera range de servidores
iptables -A FORWARD -m iprange --src-range 192.168.70.250-192.168.70.253 -j ACCEPT

# LAN → Internet (via WAN)
iptables -A FORWARD -s 192.168.70.0/24 -o eth0 -p tcp -m multiport --dports 80,443 -j ACCEPT
iptables -A FORWARD -s 192.168.70.0/24 -o eth0 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -s 192.168.70.0/24 -o eth0 -p tcp --dport 53 -j ACCEPT

# LAN -> LAN permitido (DNS 53, Kerberos 88, LDAP 389, SMB 445, RPC 135 e 49152–65535)
iptables -A FORWARD -s 192.168.70.0/24 -d 192.168.70.0/24 -j ACCEPT
```

🔐 NAT Table Rules

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 LOG of dropped packets (optional)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# Knock: records IP

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m recent --set --name KNOCK --rsource -j DROP

# SSH released ONCE and removes the knock

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m recent --rcheck --seconds 15 --name KNOCK --rsource -m recent --remove --name KNOCK --rsource -j ACCEPT

Save rules persistently:

```bash
iptables-save > /etc/iptables/iptables.rules
```

LIST the rules created (number, verbose, list):

```bash
iptables -nvL --line-number
iptables -t nat -nvL --line-number
```

```bash
cat /etc/iptables/iptables.rules
```

```bash
# Generated by iptables-save v1.8.11 on Thu Apr 23 23:45:05 2026
*nat
:PREROUTING ACCEPT [21:2280]
:INPUT ACCEPT [3:184]
:OUTPUT ACCEPT [7:556]
:POSTROUTING ACCEPT [7:556]
-A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
COMMIT
# Completed on Thu Apr 23 23:45:05 2026
# Generated by iptables-save v1.8.11 on Thu Apr 23 23:45:05 2026
*raw
:PREROUTING ACCEPT [1771:156916]
:OUTPUT ACCEPT [1746:161363]
COMMIT
# Completed on Thu Apr 23 23:45:05 2026
# Generated by iptables-save v1.8.11 on Thu Apr 23 23:45:05 2026
*mangle
:PREROUTING ACCEPT [1771:156916]
:INPUT ACCEPT [1762:156184]
:FORWARD ACCEPT [9:732]
:OUTPUT ACCEPT [1746:161363]
:POSTROUTING ACCEPT [1754:162019]
COMMIT
# Completed on Thu Apr 23 23:45:05 2026
# Generated by iptables-save v1.8.11 on Thu Apr 23 23:45:05 2026
*filter
:INPUT DROP [15:1860]
:FORWARD DROP [1:76]
:OUTPUT ACCEPT [1746:161363]
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -s 192.168.70.0/24 -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -s 192.168.70.0/24 -p tcp -m tcp --dport 22254 -j ACCEPT
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "DROP_INPUT: "
-A INPUT -i eth0 -p tcp -m tcp --dport 34567 -m conntrack --ctstate NEW -m recent --set --name KNOCK --mask 255.255.255.255 --rsource -j DROP
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i eth0 -p tcp -m tcp --dport 22254 -m conntrack --ctstate NEW -m recent --rcheck --seconds 15 --name KNOCK --mask 255.255.255.255 --rsource -m recent --remove --name KNOCK --mask 255.255.255.255 --rsource -j ACCEPT
-A INPUT -s 192.168.70.0/24 -i eth1 -p udp -m udp --dport 53 -j ACCEPT
-A INPUT -s 192.168.70.0/24 -i eth1 -p tcp -m tcp --dport 53 -j ACCEPT
-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -p icmp -j ACCEPT
-A FORWARD -m iprange --src-range 192.168.70.250-192.168.70.253 -j ACCEPT
-A FORWARD -s 192.168.70.0/24 -o eth0 -p tcp -m multiport --dports 80,443 -j ACCEPT
-A FORWARD -s 192.168.70.0/24 -o eth0 -p udp -m udp --dport 53 -j ACCEPT
-A FORWARD -s 192.168.70.0/24 -o eth0 -p tcp -m tcp --dport 53 -j ACCEPT
-A FORWARD -s 192.168.70.0/24 -d 192.168.70.0/24 -j ACCEPT
-A FORWARD -m limit --limit 5/min -j LOG --log-prefix "DROP_FORWARD: "
COMMIT
# Completed on Thu Apr 23 23:45:05 2026
```

## ✅ 9. TESTING AND VALIDATION (HOT) OF PORT KNOCKING

Monitor knock on a terminal WITHOUT FIREWALL

```bash
tcpdump -ni eth0 tcp port 34567
```

Send the knock BY THE CUSTOMER via EXTERNAL access

```bash
nc -z 192.168.122.254 12345
```

✔ SYN arrives
✔ It's DROPed
✔ Stay registered
✔ status is visible

Expected result in tcpdump

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes

14:21:14.986974 IP 99.336.74.209.58634 > 192.168.122.254.12345: Flags [S], seq 4021117238, win 64240, options [mss 1436,sackOK,TS val 2035986741 ecr 0,nop,wscale 7], length 0
14:21:14.987007 IP 192.168.122.254.12345 > 99.336.74.209.58634: Flags [R.], seq 0, ack 4021117239, win 0, length 0
^C
2 packets captured
3 packets received by filter
0 packets dropped by kernel
```

Important technical note

- The RST is sent via the TCP stack
- The package is registered by xt_recent
- Port does not respond as a service
- There is no banner or fingerprint

Validate the IP registration (in less than 15 seconds it will be deleted)

```bash
cat /proc/net/xt_recent/KNOCK
```

Expected result

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. PERFORM EXTERNAL SSH VALIDATION

Execute the knock

```bash
nc -z 192.168.122.254 34567
```

Within 15 seconds, access

```bash
ssh -p 22254 anon@192.168.122.254
```

Recommended aliases

```bash
vim ~/.bashrc
```

Content

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

Reread the file for validation

```bash
source ~/.bashrc
```

## 11. 🎉  CHECKLIST FINAL

- Invisible SSH without knock
- Single-use Knock
- Short access window
- Ban ignore knock
- Functional NAT
- Persistent firewall
- Minimal recursive DNS (Until PDC enters)

---

🎯 THAT'S ALL FOLKS!

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
