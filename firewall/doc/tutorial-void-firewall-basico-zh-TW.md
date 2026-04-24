# 🧩 VOID LINUX 教學 — 安全方案實作 — VOIDBR 實驗室

📌 防火牆 Void Linux com IPTables + 未綁定 DNS + 連接埠敲門。

---

## 🎯 本教學的目標是在 **Void Server** 上設定 **Firewall** 伺服器，充當 **主網關 (192.168.70.254)**。

---

## 🌐 1. 網路拓撲 - 功能、IP 尋址和名稱：

- 防火牆/代理：FIREWALL 192.168.70.254

- 廣域網路：`eth0` → 192.168.122.254/24（閘道：192.168.122.1）

- 區域網路：`eth1` → 192.168.70.254/24

---

## 📌 部署目標

### 輸入

-> 允許從 LAN/DMZ ping 防火牆
-> 允許 SSH 從 LAN/DMZ 存取防火牆
-> 允許透過 LAN 存取代理
-> 允許透過具有連接埠敲門的最近模組將外部 SSH 連接到防火牆

### 向前

-> 允許 LAN 存取 AD -> DNS、SMB、DHCP、NETBIOS）
-> 允許DMZ上網
-> 允許AD存取互聯網上的DNS
-> LAN需要存取網際網路上的電子郵件
-> 允許來自 LAN 的 DMZ ping
-> LAN需要存取revenuenet、gov等。

### 網路位址轉換

-> 發布 Web 伺服器 -> 80/443
-> LAN 和 DMZ 的出站 NAT

---

## ✅ 2. 網路圖

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

從另一個角度看

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



## ✅ 3. 目標與假設

- 拒絕預設策略
- 主動 IPv4 路由
- 掃描器永遠看不到門
- 防火牆作為唯一的入口點
- 受連接埠敲門保護的 SSH
- LAN 的受控 NAT
- 透過 SSH 隧道進行遠端管理

## 注意：教學在root用戶下運行！ ！

## ✅ 4.更新並安裝必要的軟體包

更新系統

```bash
vinstall -Syu
```

安裝軟體包

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

## dnsmasq 檔案的基本配置

```bash
vim /etc/unbound/unbound.conf
```

內容：

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

## 啟用未綁定

```bash
vservice enable unbound
```

## 驗證線上服務

```bash
vsv
```

## ✅ 5.SSH 配置

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

新增尖線

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## 服務啟用

```bash
vservice enable sshd
```

## 全面部署後：

- 禁用 root 登入

- 僅使用金鑰身份驗證

## ✅ 6. 防火牆網路設置

```bash
vim /etc/dhcpcd.conf
```

內容

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

申請

```bash
vservice restart dhcpcd
```

## ✅ 7. 連接埠敲擊 – 核心支持

載入所需模組

```bash
modprobe xt_recent
```

證實：

```bash
lsmod | grep xt_recent
```

預期結果

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## 透過建立檔案來持久地添加這些模組

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. 防火牆 IPTables

啟用防火牆網卡之間的路由

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

無需重啟即可應用：

```bash
sysctl --system
```

清除所有預規則（如果有）：

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 FILTER 表規則

設定預設策略：

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

輸入（防火牆存取）

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

🔐 FORWARD 表格規則（網路之間的流量）

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

🔐 NAT 表規則

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 丟包日誌（可選）

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# 敲門：記錄IP

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m 最近 --set --name KNOCK --rsource -j DROP

# SSH 釋放一次並消除敲擊

iptables -A 輸入 -m conntrack --ctstate 已建立，相關 -j 接受
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m 最近 --rcheck --秒 15 --name KNOCK --rsource -m 最近 --remove --name KNOCK --rsource -j ACCEPT

持久保存規則：

```bash
iptables-save > /etc/iptables/iptables.rules
```

列出已建立的規則（數字、詳細、清單）：

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

## ✅ 9. 端口敲擊的測試和驗證（熱）

無需防火牆即可監控終端敲擊

```bash
tcpdump -ni eth0 tcp port 34567
```

由客戶透過外部存取發送敲門聲

```bash
nc -z 192.168.122.254 12345
```

✔ SYN 到達
✔ 已刪除
✔ 保持註冊狀態
✔ 狀態可見

tcpdump 中的預期結果

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
01:04:11.194410 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197978431 ecr 0,nop,wscale 7], length 0
01:04:12.205806 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197979442 ecr 0,nop,wscale 7], length 0
```

重要技術說明

- RST 透過 TCP 堆疊發送
- 該套件由 xt_recent 註冊
- 連接埠不作為服務響應
- 沒有橫幅或指紋

驗證IP註冊（15秒內將被刪除）

```bash
cat /proc/net/xt_recent/KNOCK
```

預期結果

```bash
src=192.168.122.1 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. 執行外部 SSH 驗證

執行敲擊

```bash
nc -z 192.168.122.254 34567
```

15秒內，訪問

```bash
ssh -p 22254 anon@192.168.122.254
```

推薦別名

```bash
vim ~/.bashrc
```

內容

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

重新讀取檔案進行驗證

```bash
source ~/.bashrc
```

## 11. 🎉 最終清單

- 隱形SSH無需敲門
- 免洗敲擊器
- 訪問窗口短
- 功能性NAT
- 
- 最小遞歸 DNS（直到 PDC 進入）

---

🎯 這就是大家！

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
