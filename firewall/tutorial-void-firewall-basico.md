# 🧩 TUTORIAL VOID LINUX — IMPLANTAÇÃO DO ESQUEMA DE SEGURANÇA – LABORATÓRIO VOIDBR

📌 Firewall com Void Linux, IPTables, NAT, Port Knocking.

---

# 🔥 FIREWALL + PROXY INTEGRADO AO DOMÍNIO SAMBA4 (VOIDBR.ORG)

## 🎯 OBJETIVO nesse tutorial é Configurar um servidor **Firewall + Squid Proxy** no **Debian 13**, atuando como **gateway principal (192.168.70.254)** e integrado ao domínio **VOIDBR.ORG** (Controlador de Domínio: **192.168.70.253**), para autenticar usuários no Squid via **Active Directory (NTLM)** e aplicar **políticas de bloqueio**.

--- 

## 🌐 1. Topologia da rede - Função, endereçamento ip e nomes:

- Domínio: VOIDBR.ORG

- Firewall/Proxy: SRVFIREWALL 192.168.70.254

- Controlador de Domínio: SRVDC01 192.168.70.253

- FileServer: SRVARQUIVOS 192.168.70.252

- WAN: `eth0` → 192.168.122.254/24 (gateway: 192.168.122.1)

- LAN: `eth1` → 192.168.70.254/24

---

## 📌 Alvo da implantação

### INPUT

-> Permitir ping no Firewall vindo da LAN/DMZ
-> Permitir SSH ao Firewall vindo da LAN/DMZ
-> Permitir acesso ao Proxy pela LAN
-> Permitir SSH externo ao Firewall pelo módulo RECENT com Port knock

### FORWARD

-> Permitir LAN acessar AD -> DNS, SMB, DHCP, NETBIOS)
-> Permitir que a DMZ acesse a internet
-> Permitir que o AD acesse DNS na internet
-> LAN precisa acessar e-mail na internet
-> Permitir ping na DMZ vindo da LAN
-> LAN precisa acessar receitanet, gov, etc

### NAT

-> Publicar Servidor Web -> 80/443
-> NAT de saída para LAN e para DMZ

---

## ✅ 2. ESQUEMA DA REDE

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

Vista de outro ângulo

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

O firewall é o único host exposto à Internet.

## ✅ 3. OBJETIVOS E PREMISSAS

- Política default deny
- Roteamento IPv4 ativo
- Scanner nunca vê a porta
- Firewall como único ponto de entrada
- Nenhum painel web publicado
- SSH protegido por Port Knocking
- Controle de brute-force via Fail2ban
- NAT controlado para a LAN
- Administração remota via túnel SSH

## Observação: Tutorial rodando sob usuário root!!

## ✅ 4. ATUALIZAÇÃO E INSTALAÇÃO DOS PACOTES NECESSÁRIOS

Atualize o sistema

```bash
vinstall -Syu
```

Instale os pacotes

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

## Habilita o DNSMASQ

```bash
vservice enable dnsmasq
```

## Valida os serviços online

```bash
vsv
```

## ✅ 5. CONFIGURAÇÃO DO SSH

```bash
vim /etc/ssh/sshd_config
```

Ajustes as linhas apontadas

```bash
Port 22254
ListenAddress 0.0.0.0

PermitRootLogin yes
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban depende de log, garanta as linhas

```bash
SyslogFacility AUTH
LogLevel INFO
```

## Ativação do serviço

```bash
vservice enable sshd
```

## Após a implantação completa:

- Desabilitar login de root

- Usar apenas autenticação por chave

## ✅ 6. CONFIGURAÇÃO DE REDE DO FIREWALL

```bash
vim /etc/dhcpcd.conf
```

Conteúdo

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

Aplicar

```bash
vservice restart dhcpcd
```

## ✅ 7. PORT KNOCKING – SUPORTE EM KERNEL

Carregar o módulo necessário

```bash
modprobe xt_recent
```

Validar:

```bash
lsmod | grep xt_recent
```

Resultado esperado

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## Adicione esses módulos de modo persistente, criando o arquivo

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. FIREWALL IPTABLES

Habilite o roteamento entre as placas de rede do Firewall

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

Aplicar sem reboot:

```bash
sysctl --system
```

Limpar todas as regras prévias (se houver):

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 Regras da tabela FILTER

Setar políticas padrão:

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

INPUT (acesso ao firewall)

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

🔐 Regras da tabela FORWARD (tráfego entre redes)

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

🔐 Regras da tabela NAT

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 LOG de pacotes descartados (opcional)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# Knock: registra IP

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m recent --set --name KNOCK --rsource -j DROP

# SSH liberado UMA VEZ e remove o knock

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m recent --rcheck --seconds 15 --name KNOCK --rsource -m recent --remove --name KNOCK --rsource -j ACCEPT

Salvar as regras de modo persistente:

```bash
iptables-save > /etc/iptables/iptables.rules
```

LISTAR as regras criadas (number, verbose, list):

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

## ✅ 9. TESTE E VALIDAÇÃO (Á QUENTE) DO PORT KNOCKING

Monitorar o knock em um terminal NO FIREWALL

```bash
tcpdump -ni eth0 tcp port 34567
```

Enviar o knock PELO CLIENTE por acesso EXTERNO

```bash
nc -z 192.168.122.254 12345
```

✔ o SYN chega
✔ É DROPado
✔ Fica registrado
✔ o estado está visível

Resultado esperado no tcpdump

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

Observação técnica importante

- O RST é enviado pelo stack TCP
- O pacote é registrado pelo xt_recent
- A porta não responde como serviço
- Não há banner nem fingerprint

Validar o registro do IP (em menos de 15 segundos ele apaga)

```bash
cat /proc/net/xt_recent/KNOCK
```

Resultado esperado

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. REALIZAR A VALIDAÇÃO DO SSH EXTERNO

Executar o knock

```bash
nc -z 192.168.122.254 34567
```

Dentro de 15 segundos fazer o acesso

```bash
ssh -p 22254 anon@192.168.122.254
```

Aliases recomendados

```bash
vim ~/.bashrc
```

Conteúdo

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

Releia o arquivo para validação

```bash
source ~/.bashrc
```

## 11. 🎉  CHECKLIST FINAL

- SSH invisível sem knock
- Knock de uso único
- Janela curta de acesso
- Ban ignora knock
- NAT funcional
- Firewall persistente
- DNS recursivo mínimo (Até entrar o PDC)

---

🎯 THAT'S ALL FOLKS!

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
