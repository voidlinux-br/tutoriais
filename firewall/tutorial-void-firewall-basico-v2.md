# 🧩   VOIDBR LINUX — TUTORIAL FIREWALL HARDENED (VOIDBR LAB)

📌   Iptables + Unbound + Port Knocking

---

## 🎯   OBJETIVO

Configurar um **Firewall em Void Linux** atuando como gateway da rede:

- WAN: 192.168.122.254
- LAN: 192.168.70.254
- Controle total de tráfego
- Superfície de ataque mínima

---

## 🌐   TOPOLOGIA

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

✔ Default deny - Tudo bloqueado por padrão  
✔ Apenas tráfego necessário liberado  
✔ SSH invisível e protegido por Port Knocking  
✔ Firewall como único ponto exposto  
✔ NAT controlado  
✔ DNS local seguro via Unbound  

---

# ⚠ AVISO IMPORTANTE

Este guia altera diretamente regras de firewall, rede e acesso remoto.

Se aplicado incorretamente, você pode:

- Perder acesso SSH ao servidor
- Derrubar toda a rede
- Bloquear serviços essenciais
- Precisar de acesso físico para recuperação

---

## ❗ PRÉ-REQUISITOS

Antes de continuar, certifique-se de que:

- Você possui acesso físico ao servidor **OU** console (IPMI / KVM / VNC)
- Está executando como usuário com privilégios (`sudo`)
- Sabe restaurar configurações de rede manualmente
- Tem backup das configurações atuais

---

## 🚫 NÃO FAÇA ISSO SE

- Está em produção sem janela de manutenção
- Não sabe como reverter iptables manualmente
- Não tem acesso direto ao equipamento

---

## 💣 RISCO CONTROLADO

Todas as regras seguem o modelo:

- Política padrão: DROP
- Abertura mínima necessária
- Segurança acima de conveniência

Se algo parar de funcionar, **isso é esperado até você liberar explicitamente**.

---

## 🔒 SOBRE SEGURANÇA

- SSH ficará invisível sem o Port Knocking
- O firewall será o único ponto exposto
- Todo tráfego não permitido será descartado silenciosamente

---

## 🧠 RECOMENDAÇÃO

Antes de aplicar tudo de uma vez:

1. Execute passo a passo
2. Teste cada etapa
3. Mantenha uma sessão SSH aberta enquanto configura
4. Tenha um plano de rollback

---

## 🛑 ÚLTIMO ALERTA

Se você não entende o que cada regra faz:

👉 **PARE AQUI**

Aprenda primeiro, aplique depois.

---

Continuando, você assume total responsabilidade pela aplicação.

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

## ⚠ DISCLAIMER

Este material é fornecido **"como está"**, sem qualquer tipo de garantia, expressa ou implícita, incluindo — mas não se limitando a — garantias de funcionamento, segurança, adequação a um propósito específico ou ausência de falhas.

O uso deste tutorial é **inteiramente por sua conta e risco**.

---

## ❗ RESPONSABILIDADE

Ao aplicar qualquer comando ou configuração aqui descrita, você concorda que:

- Pode perder acesso ao sistema (incluindo SSH)
- Pode interromper serviços e conectividade de rede
- Pode causar perda de dados ou indisponibilidade do ambiente

Nenhuma garantia é oferecida de que o sistema continuará funcional após as alterações.

---

## 🚫 ISENÇÃO DE RESPONSABILIDADE

Em nenhuma circunstância o autor ou contribuidores poderão ser responsabilizados por:

- Danos diretos ou indiretos
- Perda de dados
- Falhas de segurança
- Interrupção de serviços
- Qualquer prejuízo decorrente do uso ou má aplicação deste conteúdo

---

## 🧠 USO CONSCIENTE

Este material assume que você:

- Entende o básico de redes e firewall
- Sabe recuperar o sistema manualmente
- Possui acesso físico ou console ao equipamento

Se esse não for o seu caso:

👉 **Não aplique este conteúdo em ambiente crítico**

---

## 🛑 AVISO FINAL

Se você não compreende exatamente o que cada comando faz:

**não execute.**

Estude primeiro, aplique depois.

---

Ao continuar, você declara que leu, entendeu e assume total responsabilidade pelas ações realizadas.

---
Copyright (C) 2026 VoidBR Team - https://github.com/voidlinuxbr  
👉   https://t.me/m3vorak  
👉   https://t.me/vcatafesta  



