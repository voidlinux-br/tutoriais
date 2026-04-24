# 🧩 VOID LINUX TUTORIAL — РЕАЛИЗАЦИЯ СХЕМЫ БЕЗОПАСНОСТИ — VOIDBR LABORATORY

📌 Firewall Void Linux com IPTables + Unbound DNS + Knocking Port.

---

## 🔥 БРАНДМАУЭР + ПРОКСИ, ИНТЕГРИРОВАННЫЕ В ДОМЕН SAMBA4 (VOIDBR.NET)

## 🎯 ЦЕЛЬ этого руководства — настроить сервер **Брандмауэр** на **Void Server**, действующий как **основной шлюз (192.168.70.254)**.

---

## 🌐 1. Топология сети — функции, IP-адресация и имена:

- Домен: VOIDBR.NET

- Брандмауэр/прокси: SRVFIREWALL 192.168.70.254

- Контроллер домена: SRVDC01 192.168.70.253

- Файловый сервер: SRVFILES 192.168.70.252

- WAN: `eth0` → 192.168.122.254/24 (шлюз: 192.168.122.1)

- LAN: `eth1` → 192.168.70.254/24

---

## 📌 Цель развертывания

### ВХОД

-> Разрешить пинг брандмауэру из LAN/DMZ
-> Разрешить SSH для межсетевого экрана из LAN/DMZ
-> Разрешить доступ к прокси через локальную сеть
-> Разрешить внешний SSH к брандмауэру через модуль ПОСЛЕДНИЕ с отключением порта.

### ВПЕРЕД

-> Разрешить доступ к локальной сети AD -> DNS, SMB, DHCP, NETBIOS)
-> Разрешить DMZ доступ в Интернет
-> Разрешить AD доступ к DNS в Интернете
-> Локальной сети необходим доступ к электронной почте в Интернете.
-> Разрешить пинг DMZ из локальной сети
-> ЛВС необходим доступ к сети доходов, правительству и т. д.

### НАТ

-> Публикация веб-сервера -> 80/443
-> Исходящий NAT для локальной сети и DMZ

---

## ✅ 2. СХЕМА СЕТИ

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

Взгляд под другим углом

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

Брандмауэр — единственный хост, подключенный к Интернету.

## ✅ 3. ЦЕЛИ И ПРЕДПОЛОЖЕНИЯ

- Запретить политику по умолчанию
- Активная маршрутизация IPv4
- Сканер никогда не видит дверь
- Брандмауэр как единственная точка входа
- Веб-панели не опубликованы
- SSH защищен с помощью Port Knocking
- Контроль брутфорса через Fail2ban
- Контролируемый NAT для локальной сети
- Удаленное администрирование через SSH-туннель

## Примечание. Учебное пособие выполняется под пользователем root!

## ✅ 4. ОБНОВЛЕНИЕ И УСТАНОВКА НЕОБХОДИМЫХ ПАКЕТОВ

Обновите систему

```bash
vinstall -Syu
```

Установите пакеты

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

## Базовая конфигурация файла dnsmasq

```bash
vim /etc/unbound/unbound.conf
```

Содержание:

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

## Включить НЕСВЯЗАННОЕ

```bash
vservice enable unbound
```

## Проверяет онлайн-сервисы

```bash
vsv
```

## ✅ 5. КОНФИГУРАЦИЯ SSH

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

Добавьте заостренные линии

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## Активация услуги

```bash
vservice enable sshd
```

## После полного развертывания:

- Отключить root-вход

- Используйте только ключевую аутентификацию

## ✅ 6. НАСТРОЙКА СЕТИ БРАНДМАУЭРА

```bash
vim /etc/dhcpcd.conf
```

Содержание

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

Применять

```bash
vservice restart dhcpcd
```

## ✅ 7. ВЫБОР ПОРТОВ – ПОДДЕРЖКА ЯДРА

Загрузите необходимый модуль

```bash
modprobe xt_recent
```

Подтвердить:

```bash
lsmod | grep xt_recent
```

Ожидаемый результат

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## Добавьте эти модули постоянно, создав файл

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. IPTABLES БРАНДМАУЭРА

Включить маршрутизацию между сетевыми картами брандмауэра

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

Применить без перезагрузки:

```bash
sysctl --system
```

Очистите все предварительные правила (если есть):

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 Правила таблицы ФИЛЬТР

Установите политики по умолчанию:

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

ВХОД (доступ к брандмауэру)

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

🔐 Правила таблицы FORWARD (трафик между сетями)

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

🔐 Правила таблицы NAT

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 ЖУРНАЛ отброшенных пакетов (необязательно)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# Стук: записывает IP

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m недавние --set --name KNOCK --rsource -j DROP

# SSH выпущен ОДИН РАЗ и убирает стук

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m недавние --rcheck --секунды 15 --name KNOCK --rsource -m недавние --remove --name KNOCK --rsource -j ПРИНЯТЬ

Сохраняем правила постоянно:

```bash
iptables-save > /etc/iptables/iptables.rules
```

СПИСОК созданных правил (число, подробный, список):

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

## ✅ 9. ТЕСТИРОВАНИЕ И ВАЛИДАЦИЯ (ГОРЯЧАЯ) СТУКОВКИ ПОРТОВ

Монитор стучит в терминал БЕЗ ФИРМЭРАЛА

```bash
tcpdump -ni eth0 tcp port 34567
```

Отправить стук ЗАКАЗЧИКУ через ВНЕШНИЙ доступ

```bash
nc -z 192.168.122.254 12345
```

✔ SYN прибывает
✔ Оно выпало
✔ Оставайтесь зарегистрированными
✔ статус виден

Ожидаемый результат в tcpdump

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

Важное техническое примечание

- RST отправляется через стек TCP.
- Пакет зарегистрирован xt_recent
- Порт не отвечает как услуга
- Нет баннера или отпечатка пальца

Подтвердите регистрацию IP (менее чем через 15 секунд он будет удален)

```bash
cat /proc/net/xt_recent/KNOCK
```

Ожидаемый результат

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. ВЫПОЛНИТЕ ВНЕШНЮЮ ПРОВЕРКУ SSH

Выполнить стук

```bash
nc -z 192.168.122.254 34567
```

В течение 15 секунд получите доступ

```bash
ssh -p 22254 anon@192.168.122.254
```

Рекомендуемые псевдонимы

```bash
vim ~/.bashrc
```

Содержание

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

Перечитайте файл для проверки.

```bash
source ~/.bashrc
```

## 11. 🎉 ЗАКЛЮЧИТЕЛЬНЫЙ КОНТРОЛЬНЫЙ СПИСОК

- Невидимый SSH без стука
- Одноразовый стук
- Короткое окно доступа
- Забанить игнорировать стук
- Функциональный НАТ
- Постоянный брандмауэр
- Минимальный рекурсивный DNS (до входа PDC)

---

🎯ВОТ ВСЕ, ЛЮДИ!

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
