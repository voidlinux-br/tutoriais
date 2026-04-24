# 🧩 VOID LINUX TUTORIAL – SICHERHEITSSCHEMA-IMPLEMENTIERUNG – VOIDBR-LABOR

📌 Firewall mit ungültigem Linux, IPTables, NAT, Port Knocking.

---

# 🔥 FIREWALL + PROXY IN DIE SAMBA4-DOMÄNE INTEGRIERT (VOIDBR.NET)

## 🎯 ZIEL in diesem Tutorial ist die Konfiguration eines **Firewall + Squid Proxy**-Servers auf **Debian 13**, der als **Haupt-Gateway (192.168.70.254)** fungiert und in die **VOIDBR.NET**-Domäne (Domänencontroller: **192.168.70.253**) integriert ist, um Benutzer in Squid über **Active Directory (NTLM)** zu authentifizieren und **Blockierung anzuwenden Richtlinien**.

---

## 🌐 1. Netzwerktopologie – Funktion, IP-Adressierung und Namen:

- Domain: VOIDBR.NET

- Firewall/Proxy: SRVFIREWALL 192.168.70.254

- Domänencontroller: SRVDC01 192.168.70.253

- Dateiserver: SRVFILES 192.168.70.252

- WAN: „eth0“ → 192.168.122.254/24 (Gateway: 192.168.122.1)

- LAN: „eth1“ → 192.168.70.254/24

---

## 📌 Bereitstellungsziel

### EINGANG

-> Ping an die Firewall über LAN/DMZ zulassen
-> Erlauben Sie SSH zur Firewall von LAN/DMZ aus
-> Zugriff auf Proxy über LAN zulassen
-> Erlauben Sie externes SSH zur Firewall über das RECENT-Modul mit Port Knock

### NACH VORNE

-> LAN-Zugriff zulassen (AD -> DNS, SMB, DHCP, NETBIOS)
-> Erlauben Sie DMZ den Zugriff auf das Internet
-> Erlauben Sie AD den Zugriff auf DNS im Internet
-> LAN muss auf E-Mails im Internet zugreifen
-> DMZ-Ping vom LAN zulassen
-> LAN muss Zugriff auf RevenueNet, Gov usw. haben.

### NAT

-> Webserver veröffentlichen -> 80/443
-> Ausgehendes NAT für LAN und DMZ

---

## ✅ 2. NETZWERKDIAGRAMM

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

Blick aus einem anderen Blickwinkel

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

Die Firewall ist der einzige Host, der dem Internet ausgesetzt ist.

## ✅ 3. ZIELE UND ANNAHMEN

- Standardrichtlinie verweigern
- Aktives IPv4-Routing
- Der Scanner sieht die Tür nie
- Firewall als einziger Eintrittspunkt
- Keine Web-Dashboards veröffentlicht
- SSH geschützt durch Port Knocking
- Brute-Force-Kontrolle über Fail2ban
- Kontrolliertes NAT für das LAN
- Fernverwaltung über SSH-Tunnel

## Hinweis: Tutorial läuft unter Root-Benutzer!!

## ✅ 4. ERFORDERLICHE PAKETE AKTUALISIEREN UND INSTALLIEREN

Aktualisieren Sie das System

```bash
vinstall -Syu
```

Installieren Sie die Pakete

```bash
vinstall -y \
  vim \
  bash-completion \
  iptables \
  iproute2 \
  openssh \
  tcpdump \
  conntrack-tools \
  fail2ban \
  dnsmasq
```

## Grundkonfiguration für die dnsmasq-Datei

```bash
vim /etc/dnsmasq.conf
```

Inhalt:

```bash
# Interface onde o dnsmasq vai escutar
interface=eth1
bind-interfaces

# Não usar /etc/resolv.conf automaticamente (opcional)
no-resolv

# Servidores DNS upstream
server=8.8.8.8
server=1.1.1.1

# Cache DNS
cache-size=1000

# Domínio local
domain=local
expand-hosts

# Arquivo de hosts adicional
addn-hosts=/etc/hosts

# Logs (útil para debug)
log-queries
log-dhcp
```

## Aktivieren Sie DNSMASQ

```bash
vservice enable dnsmasq
```

## Validiert Online-Dienste

```bash
vsv
```

## ✅ 5. SSH-KONFIGURATION

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

Fügen Sie die spitzen Linien hinzu

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## Dienstaktivierung

```bash
vservice enable sshd
```

## Nach vollständiger Bereitstellung:

- Deaktivieren Sie die Root-Anmeldung

- Verwenden Sie nur die Schlüsselauthentifizierung

## ✅ 6. FIREWALL-NETZWERK-SETUP

```bash
vim /etc/dhcpcd.conf
```

Inhalt

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

Anwenden

```bash
vservice restart dhcpcd
```

## ✅ 7. PORT KNOCKING – KERNEL-UNTERSTÜTZUNG

Laden Sie das erforderliche Modul

```bash
modprobe xt_recent
```

Bestätigen:

```bash
lsmod | grep xt_recent
```

Erwartetes Ergebnis

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## Fügen Sie diese Module dauerhaft hinzu, indem Sie die Datei erstellen

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. FIREWALL IPTABLES

Aktivieren Sie das Routing zwischen Firewall-Netzwerkkarten

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

Ohne Neustart anwenden:

```bash
sysctl --system
```

Alle Vorregeln löschen (falls vorhanden):

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 Tabellenregeln FILTER

Standardrichtlinien festlegen:

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

INPUT (Firewall-Zugriff)

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

🔐 FORWARD-Tabellenregeln (Verkehr zwischen Netzwerken)

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

🔐 NAT-Tabellenregeln

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 LOG der verworfenen Pakete (optional)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# Knock: zeichnet IP auf

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m letzten --set --name KNOCK --rsource -j DROP

# SSH wird EINMAL freigegeben und beseitigt das Klopfen

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m letzten --rcheck --seconds 15 --name KNOCK --rsource -m letzten --remove --name KNOCK --rsource -j ACCEPT

Regeln dauerhaft speichern:

```bash
iptables-save > /etc/iptables/iptables.rules
```

LISTE die erstellten Regeln (Anzahl, Ausführlichkeit, Liste):

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

## ✅ 9. TESTEN UND VALIDIEREN (HEISS) VON PORT KNOCKING

Überwachen Sie das Klopfen an einem Terminal OHNE FIREWALL

```bash
tcpdump -ni eth0 tcp port 34567
```

Senden Sie den Klopf DURCH DEN KUNDEN über den EXTERNEN Zugang

```bash
nc -z 192.168.122.254 12345
```

✔ SYN kommt
✔ Es ist GEFALLEN
✔ Bleiben Sie registriert
✔ Status ist sichtbar

Erwartetes Ergebnis in tcpdump

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

Wichtiger technischer Hinweis

- Der RST wird über den TCP-Stack gesendet
- Das Paket wird von xt_recent registriert
- Der Port antwortet nicht als Dienst
- Es gibt kein Banner oder Fingerabdruck

Bestätigen Sie die IP-Registrierung (in weniger als 15 Sekunden wird sie gelöscht)

```bash
cat /proc/net/xt_recent/KNOCK
```

Erwartetes Ergebnis

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. EXTERNE SSH-VALIDIERUNG DURCHFÜHREN

Führe den Klopf aus

```bash
nc -z 192.168.122.254 34567
```

Innerhalb von 15 Sekunden Zugriff

```bash
ssh -p 22254 anon@192.168.122.254
```

Empfohlene Aliase

```bash
vim ~/.bashrc
```

Inhalt

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

Lesen Sie die Datei zur Validierung erneut

```bash
source ~/.bashrc
```

## 11. 🎉 CHECKLISTE FINALE

- Unsichtbares SSH ohne Klopfen
- Einweg-Knock
- Kurzes Zugriffsfenster
- Verbot, Klopfen zu ignorieren
- Funktionelles NAT
- Permanente Firewall
- Minimales rekursives DNS (bis PDC eintritt)

---

🎯 DAS IST ALLES, LEUTE!

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
