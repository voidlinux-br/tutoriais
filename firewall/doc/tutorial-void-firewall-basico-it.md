# 🧩 TUTORIAL VOID LINUX — IMPLEMENTAZIONE DELLO SCHEMA DI SICUREZZA – LABORATORIO VOIDBR

📌 Firewall Void Linux com IPTables + DNS non associato + Port Knocking.

---

## 🎯 L'OBBIETTIVO di questo tutorial è configurare un server **Firewall** su **Void Server**, che agisca come **gateway principale (192.168.70.254)**.

---

## 🌐 1. Topologia della rete - Funzione, indirizzi IP e nomi:

- Firewall/Proxy: FIREWALL 192.168.70.254

- WAN: `eth0` → 192.168.122.254/24 (gateway: 192.168.122.1)

- LAN: `eth1` → 192.168.70.254/24

---

## 📌Obiettivo di distribuzione

### INGRESSO

-> Consenti ping al firewall da LAN/DMZ
-> Consenti SSH al firewall da LAN/DMZ
-> Consenti l'accesso al proxy su LAN
-> Consenti SSH esterno al firewall tramite il modulo RECENTE con Port knock

### INOLTRARE

-> Consenti accesso LAN AD -> DNS, SMB, DHCP, NETBIOS)
-> Consentire a DMZ di accedere a Internet
-> Consenti ad AD di accedere al DNS su Internet
-> La LAN deve accedere alla posta elettronica su Internet
-> Consenti ping DMZ dalla LAN
-> La LAN deve accedere a Revenuenet, Gov, ecc.

### NAT

-> Pubblica Server Web -> 80/443
-> NAT in uscita per LAN e per DMZ

---

## ✅ 2. SCHEMA DI RETE

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

Vista da un'altra angolazione

```bash
Internet
  |
[ Knock correto ]
  |
iptables (xt_recent libera SSH por X segundos)
  |
sshd (porta 22254)
  |
iptables (ban definitivo do IP)
```

Il firewall è l'unico host esposto a Internet.

## ✅ 3. OBIETTIVI E ASSUNZIONI

- Nega la politica predefinita
- Routing IPv4 attivo
- Lo scanner non vede mai la porta
- Firewall come unico punto di accesso
- SSH protetto da Port Knocking
- NAT controllato per la LAN
- Amministrazione remota tramite tunnel SSH

## Nota: tutorial in esecuzione con l'utente root!!

## ✅ 4. AGGIORNA E INSTALLA I PACCHETTI NECESSARI

Aggiorna il sistema

```bash
vinstall -Syu
```

Installa i pacchetti

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

## Configurazione di base per il file dnsmasq

```bash
vim /etc/unbound/unbound.conf
```

Contenuto:

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

## Abilita NON ASSOCIATO

```bash
vservice enable unbound
```

## Convalida i servizi online

```bash
vsv
```

## ✅ 5. CONFIGURAZIONE SSH

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

Aggiungi le linee appuntite

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## Attivazione del servizio

```bash
vservice enable sshd
```

## Dopo la distribuzione completa:

- Disabilita l'accesso root

- Utilizza solo l'autenticazione con chiave

## ✅ 6. CONFIGURAZIONE DELLA RETE FIREWALL

```bash
vim /etc/dhcpcd.conf
```

Contenuto

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

Fare domanda a

```bash
vservice restart dhcpcd
```

## ✅ 7. BATTUTA DEL PORTO – SUPPORTO DEL NOCCIOLO

Caricare il modulo richiesto

```bash
modprobe xt_recent
```

Convalidare:

```bash
lsmod | grep xt_recent
```

Risultato atteso

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## Aggiungi questi moduli in modo persistente creando il file

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. IPTABLES FIREWALL

Abilita il routing tra le schede di rete Firewall

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

Applica senza riavviare:

```bash
sysctl --system
```

Cancella tutte le preregole (se presenti):

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 FILTRA le regole della tabella

Imposta le policy predefinite:

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

INPUT (accesso al firewall)

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

🔐 Regole della tabella FORWARD (traffico tra reti)

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

🔐 Regole della tabella NAT

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 LOG dei pacchetti scartati (opzionale)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# Bussa: registra l'IP

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m recente --set --name KNOCK --rsource -j DROP

# SSH rilasciato UNA VOLTA e rimuove il colpo

iptables -A INPUT -m conntrack --ctstate STABILITO,RELATO -j ACCETTA
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m recente --rcheck --seconds 15 --name KNOCK --rsource -m recente --remove --name KNOCK --rsource -j ACCEPT

Salva le regole in modo persistente:

```bash
iptables-save > /etc/iptables/iptables.rules
```

ELENCA le regole create (numero, verbose, elenco):

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

## ✅ 9. TEST E VALIDAZIONE (A CALDO) DEI PORTO

Monitorare il battito su un terminale SENZA FIREWALL

```bash
tcpdump -ni eth0 tcp port 34567
```

Inviare la bussata DAL CLIENTE tramite accesso ESTERNO

```bash
nc -z 192.168.122.254 12345
```

✔ Arriva il SYN
✔ È caduto
✔ Rimani registrato
✔ lo stato è visibile

Risultato previsto in tcpdump

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
01:04:11.194410 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197978431 ecr 0,nop,wscale 7], length 0
01:04:12.205806 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197979442 ecr 0,nop,wscale 7], length 0
```

Nota tecnica importante

- L'RST viene inviato tramite lo stack TCP
- Il pacchetto è registrato da xt_recent
- La porta non risponde come servizio
- Non c'è banner o impronta digitale

Convalida la registrazione IP (in meno di 15 secondi verrà cancellata)

```bash
cat /proc/net/xt_recent/KNOCK
```

Risultato atteso

```bash
src=192.168.122.1 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. ESEGUIRE LA VALIDAZIONE SSH ESTERNA

Esegui il colpo

```bash
nc -z 192.168.122.254 34567
```

Entro 15 secondi, accesso

```bash
ssh -p 22254 anon@192.168.122.254
```

Alias consigliati

```bash
vim ~/.bashrc
```

Contenuto

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

Rileggere il file per la convalida

```bash
source ~/.bashrc
```

## 11. 🎉 CHECKLIST FINALE

- SSH invisibile senza bussare
- Bussare monouso
- Finestra di accesso breve
- NAT funzionale
- Firewall persistente
- DNS ricorsivo minimo (fino all'ingresso del PDC)

---

🎯 E' TUTTO RAGAZZE!

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
