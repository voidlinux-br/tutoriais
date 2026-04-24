# 🧩 VOIDBR LINUX — TUTORIAL FIREWALL HARDENED (VOIDBR LAB)

📌 Iptables + Unbound + Port Knocking

---

## 🎯 OBJETIVO

Configurar um **Firewall em Void Linux** atuando como:

- Gateway principal: `192.168.70.254`
- Ponto único de entrada da rede
- Controle total de tráfego (LAN / WAN / DMZ)

---

## 🌐 TOPOLOGIA

### 🔹 Estrutura

- WAN → `eth0` → 192.168.122.254/24 (GW: 192.168.122.1)
- LAN → `eth1` → 192.168.70.254/24

```
Internet
   |
[Roteador ISP]
   |
[Firewall Void]
   |
[LAN / Switch]
```

---

## 🎯 POLÍTICA DE SEGURANÇA

✔ Default deny  
✔ SSH invisível (Port Knocking)  
✔ Firewall como único ponto exposto  
✔ NAT controlado  
✔ DNS local seguro  

---

# Antes de começar

Este tutorial descreve uma **instalação manual**, utilizando configuração explícita do sistema.  
Ele **não é um instalador automático**.

## ⚠️ Leia com atenção

- Este guia **pressupõe familiaridade com Linux**, terminal e conceitos básicos de sistemas (discos, partições, boot, serviços).
- Leia **todo o tutorial antes de executar qualquer comando**.

## 🖥️ Ambiente recomendado

- **VM (VirtualBox, QEMU, KVM, etc.)** para testes e aprendizado.
- Hardware dedicado **sem dados importantes**.
- Ambiente de laboratório ou instalação consciente.

❌ **Não recomendado** para uso direto em produção sem adaptações.

## 🔐 Sobre segurança

Durante o processo de instalação, algumas configurações **priorizam praticidade**, não segurança:
- Login do usuário `root` via SSH pode ser habilitado temporariamente.
- Autenticação por senha pode estar ativa.
- Compatibilidade legada (ex: `ssh-rsa`) pode ser permitida.

👉 **Essas configurações devem ser revisadas após a instalação**, especialmente em sistemas expostos à rede.

## 🧠 Importante saber

- Execute os comandos **um a um**, conferindo a saída.
- Ajuste nomes de discos, interfaces de rede e usuários conforme o seu sistema.
- **Não copie e cole cegamente**.
- Em caso de dúvida, **pare** e revise o passo atual.

## 🛠️ Em caso de erro

Se algo der errado:
- Não reinicie às cegas.
- Verifique e refaça os passos.
- Entre novamente no sistema com `chroot`.
- Verifique GRUB, EFI e `initramfs`.

> Errar faz parte. Entender o erro é o que separa usuário de operador.
---

## Iniciar a Instalação
Entre como root
```bash
login    : root
password : voidlinux
```

## 📦 INSTALAÇÃO

### Atualizar sistema

```bash
vinstall -Syu
```

### Instalar pacotes

```bash
vinstall -y \
  vim bash-completion \
  iptables iproute2 \
  openssh tcpdump \
  conntrack-tools unbound
```

---

## 🌐 DNS — UNBOUND

```bash
vim /etc/unbound/unbound.conf
```

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

### Ativar

```bash
vservice enable unbound
vsv
```

---

## 🔐 SSH HARDENING

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

```bash
vservice enable sshd
```

⚠ Depois:
- Desabilitar root
- Usar chave SSH

---

## 🌐 REDE DO FIREWALL

```bash
vim /etc/dhcpcd.conf
```

```bash
# WAN
interface eth0
static ip_address=192.168.122.254/24
static routers=192.168.122.1
static domain_name_servers=192.168.122.1 8.8.8.8

# LAN
interface eth1
static ip_address=192.168.70.254/24
nogateway
```

```bash
vservice restart dhcpcd
```

---

## 🧠 PORT KNOCKING (KERNEL)

```bash
modprobe xt_recent
lsmod | grep xt_recent
```

Persistência:

```bash
echo 'xt_recent' > /etc/modules-load.d/xt_recent.conf
```

---

## 🔥 IPTABLES

### Ativar roteamento

```bash
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.conf
sysctl --system
```

### Limpar regras

```bash
iptables -F
iptables -t nat -F
iptables -X
```

---

## 🔐 POLÍTICAS

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

---

## INPUT

```bash
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

# Ping LAN
iptables -A INPUT -s 192.168.70.0/24 -p icmp -j ACCEPT

# DNS LAN
iptables -A INPUT -i eth1 -p udp --dport 53 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 53 -j ACCEPT

# SSH LAN
iptables -A INPUT -s 192.168.70.0/24 -p tcp --dport 22254 -j ACCEPT
```

---

## FORWARD

```bash
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -p icmp -j ACCEPT

# LAN → Internet
iptables -A FORWARD -s 192.168.70.0/24 -o eth0 -p tcp -m multiport --dports 80,443 -j ACCEPT

# DNS
iptables -A FORWARD -s 192.168.70.0/24 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -s 192.168.70.0/24 -p tcp --dport 53 -j ACCEPT

# LAN interna
iptables -A FORWARD -s 192.168.70.0/24 -d 192.168.70.0/24 -j ACCEPT
```

---

## NAT

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

---

## 🕵️ PORT KNOCK

```bash
# Registra knock
iptables -A INPUT -i eth0 -p tcp --dport 34567 \
-m recent --set --name KNOCK --rsource -j DROP

# Libera SSH
iptables -A INPUT -i eth0 -p tcp --dport 22254 \
-m recent --rcheck --seconds 15 --name KNOCK --rsource \
-m recent --remove --name KNOCK --rsource -j ACCEPT
```

---

## 💾 SALVAR

```bash
iptables-save > /etc/iptables/iptables.rules
```

---

## 🧪 TESTE

### Monitorar

```bash
tcpdump -ni eth0 tcp port 34567
```

### Enviar knock

```bash
nc -z 192.168.122.254 34567
```

### Conectar

```bash
ssh -p 22254 anon@192.168.122.254
```

---

## ⚡ ALIASES (opcional)

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

---

## ✅ CHECKLIST FINAL

✔ SSH invisível  
✔ Knock funcional  
✔ NAT ok  
✔ DNS ativo  
✔ Firewall persistente  

---

🎯 Done.

## ⚖️ Disclaimer (Aviso Legal)

ESTE SOFTWARE/TUTORIAL É FORNECIDO "COMO ESTÁ", SEM ABSOLUTAMENTE NENHUMA GARANTIA
DE QUALQUER TIPO, EXPRESSA OU IMPLÍCITA, INCLUINDO, MAS NÃO SE LIMITANDO A,
GARANTIAS DE COMERCIALIZAÇÃO OU ADEQUAÇÃO A UM PROPÓSITO ESPECÍFICO.

O USO DESTE SOFTWARE/TUTORIAL É DE TOTAL RESPONSABILIDADE DO USUÁRIO.

EM NENHUM MOMENTO O AUTOR OU OS CONTRIBUIDORES SERÃO RESPONSÁVEIS POR
QUALQUER DANO, PERDA DE DADOS OU FALHAS NO SISTEMA DECORRENTES DO USO
DESTE PROGRAMA.

---

Copyright (C) 2026 VoidBR Team - https://github.com/voidlinuxbr  
👉   https://t.me/m3vorak  
👉   https://t.me/vcatafesta  



