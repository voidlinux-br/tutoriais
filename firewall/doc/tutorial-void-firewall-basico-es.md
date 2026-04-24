# 🧩 TUTORIAL VOID LINUX — IMPLEMENTACIÓN DEL ESQUEMA DE SEGURIDAD – LABORATORIO VOIDBR

📌 Firewall con Void Linux, IPTables, NAT, Port Knocking.

---

# 🔥 FIREWALL + PROXY INTEGRADO AL DOMINIO SAMBA4 (VOIDBR.NET)

## 🎯 El OBJETIVO de este tutorial es Configurar un servidor **Firewall + Squid Proxy** en **Debian 13**, que actúa como **puerta de enlace principal (192.168.70.254)** e integrado en el dominio **VOIDBR.NET** (Controlador de dominio: **192.168.70.253**), para autenticar usuarios en Squid a través de **Active Directory (NTLM)** y aplicar **bloqueo. políticas**.

---

## 🌐 1. Topología de red: función, direcciones IP y nombres:

- Dominio: VOIDBR.NET

- Cortafuegos/Proxy: SRVFIREWALL 192.168.70.254

- Controlador de dominio: SRVDC01 192.168.70.253

- Servidor de archivos: SRVFILES 192.168.70.252

- WAN: `eth0` → 192.168.122.254/24 (puerta de enlace: 192.168.122.1)

- LAN: `eth1` → 192.168.70.254/24

---

## 📌 Objetivo de implementación

### APORTE

-> Permitir ping al Firewall desde LAN/DMZ
-> Permitir SSH al Firewall desde LAN/DMZ
-> Permitir el acceso a Proxy a través de LAN
-> Permitir SSH externo al Firewall a través del módulo RECIENTE con Port knock

### ADELANTE

-> Permitir acceso LAN AD -> DNS, SMB, DHCP, NETBIOS)
-> Permitir que DMZ acceda a Internet
-> Permitir que AD acceda a DNS en Internet
-> LAN necesita acceder al correo electrónico en Internet
-> Permitir ping DMZ desde LAN
-> LAN necesita acceder a Revennet, gov, etc.

### NAT

-> Publicar servidor web -> 80/443
-> NAT saliente para LAN y DMZ

---

## ✅ 2. DIAGRAMA DE RED

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

Vista desde otro ángulo

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

El firewall es el único host expuesto a Internet.

## ✅ 3. OBJETIVOS Y SUPUESTOS

- Denegar política predeterminada
- Enrutamiento IPv4 activo
- El escáner nunca ve la puerta
- Firewall como único punto de entrada
- No se han publicado paneles web
- SSH protegido por Port Knocking
- Control de fuerza bruta a través de Fail2ban
- NAT controlada para la LAN
- Administración remota a través de túnel SSH

## Nota: ¡¡Tutorial que se ejecuta con el usuario root!!

## ✅ 4. ACTUALIZAR E INSTALAR LOS PAQUETES NECESARIOS

Actualizar el sistema

```bash
vinstall -Syu
```

Instalar los paquetes

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

## Configuración básica para el archivo dnsmasq

```bash
vim /etc/dnsmasq.conf
```

Contenido:

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

## Habilitar DNSMASQ

```bash
vservice enable dnsmasq
```

## Valida servicios en línea

```bash
vsv
```

## ✅ 5. CONFIGURACIÓN SSH

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

Agrega las líneas puntiagudas

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## Activación del servicio

```bash
vservice enable sshd
```

## Después de la implementación completa:

- Deshabilitar el inicio de sesión raíz

- Utilice solo autenticación de clave

## ✅ 6. CONFIGURACIÓN DE LA RED FIREWALL

```bash
vim /etc/dhcpcd.conf
```

Contenido

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

## ✅ 7. LLAMADO DE PUERTOS – SOPORTE DEL NÚCLEO

Cargue el módulo requerido

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

## Agregue estos módulos de manera persistente creando el archivo

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. IPTABLES DE CORTAFUEGOS

Habilitar el enrutamiento entre tarjetas de red Firewall

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

Aplicar sin reiniciar:

```bash
sysctl --system
```

Borre todas las reglas previas (si las hay):

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 Reglas de la tabla FILTRAR

Establecer políticas predeterminadas:

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

ENTRADA (acceso al firewall)

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

🔐 Reglas de la tabla FORWARD (tráfico entre redes)

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

🔐 Reglas de la tabla NAT

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 REGISTRO de paquetes descartados (opcional)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# Golpe: registra IP

iptables -A ENTRADA -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NUEVO -m reciente --set --name KNOCK --rsource -j DROP

# SSH se lanzó UNA VEZ y elimina el golpe

iptables -A ENTRADA -m conntrack --ctstate ESTABLECIDO,RELACIONADO -j ACEPTAR
iptables -A ENTRADA -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NUEVO -m reciente --rcheck --segundos 15 --name KNOCK --rsource -m reciente --remove --name KNOCK --rsource -j ACEPTAR

Guardar reglas de forma persistente:

```bash
iptables-save > /etc/iptables/iptables.rules
```

LISTA las reglas creadas (número, detallado, lista):

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

## ✅ 9. PRUEBAS Y VALIDACIÓN (EN CALIENTE) DEL GOLPE DE PUERTOS

Monitor de golpe en un terminal SIN FIREWALL

```bash
tcpdump -ni eth0 tcp port 34567
```

Enviar el golpe POR EL CLIENTE vía acceso EXTERNO

```bash
nc -z 192.168.122.254 12345
```

✔ Llega SYN
✔ Está CAÍDO
✔ Manténgase registrado
✔ el estado es visible

Resultado esperado en tcpdump

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

Nota técnica importante

- El RST se envía a través de la pila TCP.
- El paquete está registrado por xt_recent
- El puerto no responde como un servicio.
- No hay pancarta ni huella digital.

Validar el registro de IP (en menos de 15 segundos se eliminará)

```bash
cat /proc/net/xt_recent/KNOCK
```

Resultado esperado

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. REALIZAR VALIDACIÓN SSH EXTERNA

ejecutar el golpe

```bash
nc -z 192.168.122.254 34567
```

En 15 segundos, acceda

```bash
ssh -p 22254 anon@192.168.122.254
```

Alias recomendados

```bash
vim ~/.bashrc
```

Contenido

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

Vuelva a leer el archivo para su validación.

```bash
source ~/.bashrc
```

## 11. 🎉 LISTA DE VERIFICACIÓN FINAL

- SSH invisible sin golpes
- Golpe de un solo uso
- Ventana de acceso corto
- Prohibir ignorar el golpe
- NAT funcional
- Cortafuegos persistente
- DNS recursivo mínimo (Hasta que entre PDC)

---

🎯 ¡ESO ES TODO AMIGOS!

👉https://t.me/m3vorak
👉https://t.me/vcatafesta
