# 🧩 TUTORIAL VOID LINUX — IMPLANTAÇÃO DO ESQUEMA DE SEGURANÇA – LABORATÓRIO VOIDBR

📌 Firewall Void Linux com Iptables + Unbound DNS + Port Knocking.

---

## 🎯 OBJETIVO nesse tutorial é Configurar um servidor **Firewall** no **Void Server**, atuando como **gateway principal (192.168.70.254)**.

--- 

## 🌐 1. Topologia da rede - Função, endereçamento ip e nomes:

- Firewall/Proxy: FIREWALL 192.168.70.254

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
iptables (ban definitivo do IP)
```

O firewall é o único host exposto à Internet.

## ✅ 3. OBJETIVOS E PREMISSAS

- Política default deny
- Roteamento IPv4 ativo
- Scanner nunca vê a porta
- Firewall como único ponto de entrada
- SSH protegido por Port Knocking
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
  unbound
```

## Configuração básica para o arquivo do dnsmasq

```bash
vim /etc/unbound/unbound.conf
```

Conteúdo:

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

## Habilita o UNBOUND

```bash
vservice enable unbound
```

## Valida os serviços online

```bash
vsv
```

## ✅ 5. CONFIGURAÇÃO DO SSH

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

Adicione as linhas apontadas

```bash
Port 22254
PermitRootLogin yes
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

iptables -A INPUT -i eth1 -p udp --dport 53 -s 192.168.70.0/24 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 53 -s 192.168.70.0/24 -j ACCEPT


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
iptables -A FORWARD -s 192.168.70.0/24 -o eth0 -p tcp -m multiport --dports 80,443 -j ACCEPT
iptables -A FORWARD -s 192.168.70.0/24 -o eth0 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -s 192.168.70.0/24 -o eth0 -p tcp --dport 53 -j ACCEPT

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
01:04:11.194410 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197978431 ecr 0,nop,wscale 7], length 0
01:04:12.205806 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197979442 ecr 0,nop,wscale 7], length 0
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
src=192.168.122.1 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
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
- NAT funcional
- Firewall persistente
- DNS recursivo mínimo (Até entrar o PDC)

---

🎯 THAT'S ALL FOLKS!

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
