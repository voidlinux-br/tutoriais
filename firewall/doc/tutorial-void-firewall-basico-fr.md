# 🧩 TUTORIEL VOID LINUX — MISE EN ŒUVRE DU SYSTÈME DE SÉCURITÉ – LABORATOIRE VOIDBR

📌 Pare-feu avec Void Linux, IPTables, NAT, Port Knocking.

---

# 🔥 FIREWALL + PROXY INTÉGRÉ AU DOMAINE SAMBA4 (VOIDBR.ORG)

## 🎯 L'OBJECTIF de ce tutoriel est de configurer un serveur **Firewall + Squid Proxy** sur **Debian 13**, faisant office de **passerelle principale (192.168.70.254)** et intégré au domaine **VOIDBR.ORG** (Contrôleur de domaine : **192.168.70.253**), pour authentifier les utilisateurs dans Squid via **Active Directory (NTLM)** et appliquer le **blocage politiques**.

---

## 🌐 1. Topologie du réseau – Fonction, adressage IP et noms :

- Domaine : VOIDBR.ORG

- Pare-feu/Proxy : SRVFIREWALL 192.168.70.254

- Contrôleur de domaine : SRVDC01 192.168.70.253

- Serveur de fichiers : SRVFILES 192.168.70.252

- WAN : `eth0` → 192.168.122.254/24 (passerelle : 192.168.122.1)

- Réseau local : `eth1` → 192.168.70.254/24

---

## 📌 Cible de déploiement

### SAISIR

-> Autoriser le ping vers le pare-feu depuis LAN/DMZ
-> Autoriser SSH au pare-feu depuis LAN/DMZ
-> Autoriser l'accès au proxy sur LAN
-> Autoriser le SSH externe au Firewall via le module RECENT avec Port knock

### AVANT

-> Autoriser l'accès LAN AD -> DNS, SMB, DHCP, NETBIOS)
-> Autoriser DMZ à accéder à Internet
-> Autoriser AD à accéder au DNS sur Internet
-> Le réseau local doit accéder au courrier électronique sur Internet
-> Autoriser le ping DMZ depuis le LAN
-> Le réseau local doit accéder à Revenuenet, gov, etc.

### NAT

-> Publier le serveur Web -> 80/443
-> NAT sortant pour LAN et pour DMZ

---

## ✅ 2. SCHÉMA DE RÉSEAU

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

Vue sous un autre angle

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

Le pare-feu est le seul hôte exposé à Internet.

## ✅ 3. OBJECTIFS ET HYPOTHÈSES

- Refuser la stratégie par défaut
- Routage IPv4 actif
- Le scanner ne voit jamais la porte
- Le pare-feu comme seul point d'entrée
- Aucun tableau de bord Web publié
- SSH protégé par Port Knocking
- Contrôle par force brute via Fail2ban
- NAT contrôlé pour le LAN
- Administration à distance via tunnel SSH

## Remarque : Tutoriel exécuté sous l'utilisateur root !!

## ✅ 4. METTRE À JOUR ET INSTALLER LES FORFAITS NÉCESSAIRES

Mettre à jour le système

```bash
vinstall -Syu
```

Installer les paquets

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

## Activer DNSMASQ

```bash
vservice enable dnsmasq
```

## Valide les services en ligne

```bash
vsv
```

## ✅ 5. CONFIGURATION SSH

```bash
vim /etc/ssh/sshd_config
```

Ajustez les lignes pointues

```bash
Port 22254
ListenAddress 0.0.0.0

PermitRootLogin yes
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban dépend du journal, garantissez les lignes

```bash
SyslogFacility AUTH
LogLevel INFO
```

## Activation des services

```bash
vservice enable sshd
```

## Après le déploiement complet :

- Désactiver la connexion root

- Utiliser uniquement l'authentification par clé

## ✅ 6. CONFIGURATION DU RÉSEAU PARE-FEU

```bash
vim /etc/dhcpcd.conf
```

Contenu

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

Appliquer

```bash
vservice restart dhcpcd
```

## ✅ 7. PORT KNOCKING – SUPPORT DU NOYAU

Charger le module requis

```bash
modprobe xt_recent
```

Valider:

```bash
lsmod | grep xt_recent
```

Résultat attendu

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## Ajoutez ces modules de manière persistante en créant le fichier

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. PARE-FEU IPTABLES

Activer le routage entre les cartes réseau du pare-feu

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

Appliquer sans redémarrage :

```bash
sysctl --system
```

Effacez toutes les pré-règles (le cas échéant) :

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 FILTRER les règles de la table

Définir des politiques par défaut :

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

INPUT (accès au pare-feu)

```bash
# Libera loopback
iptables -A INPUT -i lo -j ACCEPT

# Conexões estabelecidas
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

# ICMP (ping) da LAN
iptables -A INPUT -s 192.168.70.0/24 -p icmp --icmp-type echo-request -j ACCEPT


# SSH (porta customizada)
iptables -A INPUT -s 192.168.70.0/24 -p tcp --dport 22254 -j ACCEPT
```

🔐 Règles de la table FORWARD (trafic entre réseaux)

```bash
# Conexões estabelecidas
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

# ICMP (opcional, recomendado)
iptables -A FORWARD -p icmp -j ACCEPT

# Libera range de servidores
iptables -A FORWARD -m iprange --src-range 192.168.70.250-192.168.70.253 -j ACCEPT

# LAN → Internet (via WAN)
iptables -A FORWARD -s 192.168.70.0/24 -o enp1s0 -p tcp -m multiport --dports 80,443 -j ACCEPT
iptables -A FORWARD -s 192.168.70.0/24 -o enp1s0 -p udp --dport 53 -j ACCEPT

# LAN -> LAN permitido (DNS 53, Kerberos 88, LDAP 389, SMB 445, RPC 135 e 49152–65535)
iptables -A FORWARD -s 192.168.70.0/24 -d 192.168.70.0/24 -j ACCEPT
```

🔐 Règles des tables NAT

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 LOG des paquets abandonnés (facultatif)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# Knock : enregistre l'IP

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m recent --set --name KNOCK --rsource -j DROP

# SSH libéré UNE FOIS et supprime le coup

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m récent --rcheck --seconds 15 --name KNOCK --rsource -m recent --remove --name KNOCK --rsource -j ACCEPT

Enregistrez les règles de manière persistante :

```bash
iptables-save > /etc/iptables/iptables.rules
```

LISTE les règles créées (numéro, verbeux, liste) :

```bash
iptables -nvL --line-number
iptables -t nat -nvL --line-number
```

```bash
cat /etc/iptables/iptables.rules
```

```bash
# Generated by iptables-save v1.8.11 on Thu Apr 23 19:50:45 2026
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [139:12768]
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -s 192.168.70.0/24 -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -s 192.168.70.0/24 -p tcp -m tcp --dport 22254 -j ACCEPT
-A INPUT -i eth0 -p tcp -m tcp --dport 34567 -m conntrack --ctstate NEW -m recent --set --name KNOCK --mask 255.255.255.255 --rsource -j DROP
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i eth0 -p tcp -m tcp --dport 22254 -m conntrack --ctstate NEW -m recent --rcheck --seconds 15 --name KNOCK --mask 255.255.255.255 --rsource -m recent --remove --name KNOCK --mask 255.255.255.255 --rsource -j ACCEPT
-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -p icmp -j ACCEPT
-A FORWARD -m iprange --src-range 192.168.70.250-192.168.70.253 -j ACCEPT
-A FORWARD -s 192.168.70.0/24 -o enp1s0 -p tcp -m multiport --dports 80,443 -j ACCEPT
-A FORWARD -s 192.168.70.0/24 -o enp1s0 -p udp -m udp --dport 53 -j ACCEPT
-A FORWARD -s 192.168.70.0/24 -d 192.168.70.0/24 -j ACCEPT
COMMIT
# Completed on Thu Apr 23 19:50:45 2026
# Generated by iptables-save v1.8.11 on Thu Apr 23 19:50:45 2026
*mangle
:PREROUTING ACCEPT [138:12076]
:INPUT ACCEPT [138:12076]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [139:12768]
:POSTROUTING ACCEPT [139:12768]
COMMIT
# Completed on Thu Apr 23 19:50:45 2026
# Generated by iptables-save v1.8.11 on Thu Apr 23 19:50:45 2026
*raw
:PREROUTING ACCEPT [138:12076]
:OUTPUT ACCEPT [139:12768]
COMMIT
# Completed on Thu Apr 23 19:50:45 2026
# Generated by iptables-save v1.8.11 on Thu Apr 23 19:50:45 2026
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
COMMIT
# Completed on Thu Apr 23 19:50:45 2026
```

## ✅ 9. TESTS ET VALIDATION (À CHAUD) DU PORT KNOCKING

Surveiller frapper sur un terminal SANS PARE-FEU

```bash
tcpdump -ni eth0 tcp port 34567
```

Envoyer le coup PAR LE CLIENT via un accès EXTERNE

```bash
nc -z 192.168.122.254 12345
```

✔ SYN arrive
✔ C'est abandonné
✔ Restez inscrit
✔ le statut est visible

Résultat attendu dans tcpdump

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

Note technique importante

- Le RST est envoyé via la pile TCP
- Le package est enregistré par xt_recent
- Le port ne répond pas en tant que service
- Il n'y a pas de bannière ni d'empreinte digitale

Validez l'enregistrement IP (en moins de 15 secondes il sera supprimé)

```bash
cat /proc/net/xt_recent/KNOCK
```

Résultat attendu

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. EFFECTUER UNE VALIDATION SSH EXTERNE

Exécuter le coup

```bash
nc -z 192.168.122.254 34567
```

En 15 secondes, accédez

```bash
ssh -p 22254 anon@192.168.122.254
```

Alias recommandés

```bash
vim ~/.bashrc
```

Contenu

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

Relisez le fichier pour validation

```bash
source ~/.bashrc
```

## 11. 🎉 LISTE DE CONTRÔLE FINALE

- SSH invisible sans frapper
- Coup à usage unique
- Fenêtre d'accès courte
- Interdire d'ignorer frapper
- NAT fonctionnel
- Pare-feu persistant
- DNS récursif minimal (jusqu'à ce que PDC entre)

---

🎯 C'EST TOUS LES GENS !

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
