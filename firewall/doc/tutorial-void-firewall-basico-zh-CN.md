# 🧩 VOID LINUX 教程 — 安全方案实施 — VOIDBR 实验室

📌 防火墙 com Void Linux、IPTables、NAT、端口敲门。

---

# 🔥 防火墙 + 代理集成到 SAMBA4 域 (VOIDBR.NET)

## 🎯 本教程的目标是在 **Debian 13** 上配置 **防火墙 + Squid 代理** 服务器，充当 **主网关 (192.168.70.254)** 并集成到 **VOIDBR.NET** 域（域控制器：**192.168.70.253**），通过 **Active Directory (NTLM)** 对 Squid 中的用户进行身份验证并应用 **阻止策略**。

---

## 🌐 1. 网络拓扑 - 功能、IP 寻址和名称：

- 域名：VOIDBR.NET

- 防火墙/代理：SRVFIREWALL 192.168.70.254

- 域控制器：SRVDC01 192.168.70.253

- 文件服务器：SRVFILES 192.168.70.252

- 广域网：`eth0` → 192.168.122.254/24（网关：192.168.122.1）

- 局域网：`eth1` → 192.168.70.254/24

---

## 📌 部署目标

### 输入

-> 允许从 LAN/DMZ ping 防火墙
-> 允许 SSH 从 LAN/DMZ 访问防火墙
-> 允许通过 LAN 访问代理
-> 允许通过带有端口敲门的最近模块将外部 SSH 连接到防火墙

### 向前

-> 允许 LAN 访问 AD -> DNS、SMB、DHCP、NETBIOS）
-> 允许DMZ访问互联网
-> 允许AD访问互联网上的DNS
-> LAN需要访问互联网上的电子邮件
-> 允许来自 LAN 的 DMZ ping
-> LAN需要访问revenuenet、gov等。

### 网络地址转换

-> 发布 Web 服务器 -> 80/443
-> LAN 和 DMZ 的出站 NAT

---

## ✅ 2. 网络图

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

从另一个角度看

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

防火墙是唯一暴露于互联网的主机。

## ✅ 3. 目标和假设

- 拒绝默认策略
- 主动 IPv4 路由
- 扫描仪永远看不到门
- 防火墙作为唯一的入口点
- 没有发布网络仪表板
- 受端口敲门保护的 SSH
- 通过 Fail2ban 进行暴力控制
- LAN 的受控 NAT
- 通过 SSH 隧道进行远程管理

## 注意：教程在root用户下运行！！

## ✅ 4.更新并安装必要的软件包

更新系统

```bash
vinstall -Syu
```

安装软件包

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

## dnsmasq 文件的基本配置

```bash
vim /etc/dnsmasq.conf
```

内容：

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

## 启用 DNSMASQ

```bash
vservice enable dnsmasq
```

## 验证在线服务

```bash
vsv
```

## ✅ 5.SSH 配置

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

添加尖线

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## 服务激活

```bash
vservice enable sshd
```

## 全面部署后：

- 禁用 root 登录

- 仅使用密钥身份验证

## ✅ 6. 防火墙网络设置

```bash
vim /etc/dhcpcd.conf
```

内容

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

申请

```bash
vservice restart dhcpcd
```

## ✅ 7. 端口敲击 – 内核支持

加载所需模块

```bash
modprobe xt_recent
```

证实：

```bash
lsmod | grep xt_recent
```

预期结果

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## 通过创建文件来持久添加这些模块

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. 防火墙 IPTables

启用防火墙网卡之间的路由

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

无需重启即可应用：

```bash
sysctl --system
```

清除所有预规则（如果有）：

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 FILTER 表规则

设置默认策略：

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

输入（防火墙访问）

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

🔐 FORWARD 表规则（网络之间的流量）

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

🔐 NAT 表规则

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 丢包日志（可选）

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# 敲门：记录IP

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m 最近 --set --name KNOCK --rsource -j DROP

# SSH 释放一次并消除敲击

iptables -A 输入 -m conntrack --ctstate 已建立，相关 -j 接受
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m 最近 --rcheck --秒 15 --name KNOCK --rsource -m 最近 --remove --name KNOCK --rsource -j ACCEPT

持久保存规则：

```bash
iptables-save > /etc/iptables/iptables.rules
```

列出创建的规则（数字、详细、列表）：

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

## ✅ 9. 端口敲击的测试和验证（热）

无需防火墙即可监控终端敲击

```bash
tcpdump -ni eth0 tcp port 34567
```

由客户通过外部访问发送敲门声

```bash
nc -z 192.168.122.254 12345
```

✔ SYN 到达
✔ 已删除
✔ 保持注册状态
✔ 状态可见

tcpdump 中的预期结果

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

重要技术说明

- RST 通过 TCP 堆栈发送
- 该包由 xt_recent 注册
- 端口不作为服务响应
- 没有横幅或指纹

验证IP注册（15秒内将被删除）

```bash
cat /proc/net/xt_recent/KNOCK
```

预期结果

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. 执行外部 SSH 验证

执行敲击

```bash
nc -z 192.168.122.254 34567
```

15秒内，访问

```bash
ssh -p 22254 anon@192.168.122.254
```

推荐别名

```bash
vim ~/.bashrc
```

内容

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

重新读取文件进行验证

```bash
source ~/.bashrc
```

## 11. 🎉 最终清单

- 隐形SSH无需敲门
- 一次性敲击器
- 访问窗口短
- 禁止无视敲门
- 功能性NAT
- 持久防火墙
- 最小递归 DNS（直到 PDC 进入）

---

🎯 这就是大家！

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
