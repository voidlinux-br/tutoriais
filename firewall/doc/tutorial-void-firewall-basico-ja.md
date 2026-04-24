# 🧩 VOID LINUX チュートリアル — セキュリティ スキームの実装 – VOIDBR LABORATORY

📌 ファイアウォール Linux com IPTables + アンバインド DNS + ポートノッキングを無効にします。

---

## 🎯 このチュートリアルの目的は、**Void サーバー** 上に **メイン ゲートウェイ (192.168.70.254)** として機能する **ファイアウォール** サーバーを構成することです。

---

## 🌐 1. ネットワーク トポロジ - 機能、IP アドレス指定および名前:

- ファイアウォール/プロキシ: ファイアウォール 192.168.70.254

- WAN: `eth0` → 192.168.122.254/24 (ゲートウェイ: 192.168.122.1)

- LAN: `eth1` → 192.168.70.254/24

---

## 📌 導入対象

### 入力

-> LAN/DMZ からファイアウォールへの ping を許可する
-> LAN/DMZ からファイアウォールへの SSH を許可する
-> LAN 経由のプロキシへのアクセスを許可する
-> ポートノックを使用して、RECENT モジュール経由でファイアウォールへの外部 SSH を許可します。

### フォワード

-> LAN アクセスを許可 AD -> DNS、SMB、DHCP、NETBIOS)
-> DMZ がインターネットにアクセスできるようにする
-> AD がインターネット上の DNS にアクセスできるようにする
-> LAN はインターネット上の電子メールにアクセスする必要があります
-> LAN からの DMZ ping を許可する
-> LAN は収益ネット、政府などにアクセスする必要があります。

### NAT

-> Web サーバーの公開 -> 80/443
-> LAN および DMZ のアウトバウンド NAT

---

## ✅ 2. ネットワーク図

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

別の角度から見る

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

ファイアウォールは、インターネットに公開される唯一のホストです。

## ✅ 3. 目的と前提

- デフォルトポリシーを拒否する
- アクティブなIPv4ルーティング
- スキャナーはドアを決して認識しません
- 唯一のエントリポイントとしてのファイアウォール
- ポートノッキングで保護されたSSH
- LAN 用の制御された NAT
- SSHトンネル経由のリモート管理

## 注: チュートリアルは root ユーザーで実行されます。

## ✅ 4. 必要なパッケージを更新してインストールする

システムをアップデートする

```bash
vinstall -Syu
```

パッケージをインストールする

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

## dnsmasq ファイルの基本構成

```bash
vim /etc/unbound/unbound.conf
```

コンテンツ：

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

## UNBOUNDを有効にする

```bash
vservice enable unbound
```

## オンラインサービスを検証します

```bash
vsv
```

## ✅ 5. SSH 設定

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

尖った線を追加します

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## サービスのアクティベーション

```bash
vservice enable sshd
```

## 完全な展開後:

- rootログインを無効にする

- キー認証のみを使用する

## ✅ 6. ファイアウォールネットワークのセットアップ

```bash
vim /etc/dhcpcd.conf
```

コンテンツ

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

適用する

```bash
vservice restart dhcpcd
```

## ✅ 7. ポートノッキング – カーネルサポート

必要なモジュールをロードします

```bash
modprobe xt_recent
```

検証:

```bash
lsmod | grep xt_recent
```

期待される結果

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ファイルを作成してこれらのモジュールを永続的に追加します。

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. ファイアウォール IP テーブル

ファイアウォールネットワークカード間のルーティングを有効にする

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

再起動せずに適用します。

```bash
sysctl --system
```

すべての事前ルール (存在する場合) をクリアします。

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 FILTER テーブルのルール

デフォルトのポリシーを設定します。

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

INPUT (ファイアウォールアクセス)

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

🔐 FORWARD テーブル ルール (ネットワーク間のトラフィック)

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

🔐 NAT テーブルのルール

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 ドロップされたパケットのログ (オプション)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# ノック：IPを記録します

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m Recent --set --name KNOCK --rsource -j DROP

# SSH が 1 回解放され、ノックが解除されます

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m 最近の --rcheck --秒 15 --name KNOCK --rsource -m 最近の --remove --name KNOCK --rsource -j ACCEPT

ルールを永続的に保存します。

```bash
iptables-save > /etc/iptables/iptables.rules
```

作成されたルールをリストします (数値、詳細、リスト):

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

## ✅ 9. ポートノッキングのテストと検証 (ホット)

ファイアウォールなしで端末のノックを監視する

```bash
tcpdump -ni eth0 tcp port 34567
```

顧客が外部アクセス経由でノックを送信します

```bash
nc -z 192.168.122.254 12345
```

✔ SYNが到着
✔ 削除されました
✔ 登録を続ける
✔ステータスが見える

tcpdump で期待される結果

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
01:04:11.194410 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197978431 ecr 0,nop,wscale 7], length 0
01:04:12.205806 IP 192.168.122.1.33886 > 192.168.122.254.34567: Flags [S], seq 740762705, win 64240, options [mss 1460,sackOK,TS val 197979442 ecr 0,nop,wscale 7], length 0
```

重要な技術上の注意事項

- RST は TCP スタック経由で送信されます
- パッケージは xt_recent によって登録されます
- ポートがサービスとして応答しない
- バナーや指紋はありません

IP 登録を検証します (15 秒以内に削除されます)。

```bash
cat /proc/net/xt_recent/KNOCK
```

期待される結果

```bash
src=192.168.122.1 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. 外部 SSH 検証を実行する

ノックを実行する

```bash
nc -z 192.168.122.254 34567
```

15秒以内にアクセスしてください

```bash
ssh -p 22254 anon@192.168.122.254
```

推奨されるエイリアス

```bash
vim ~/.bashrc
```

コンテンツ

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

検証のためにファイルを再読み込みします

```bash
source ~/.bashrc
```

## 11. 🎉 最終チェックリスト

- ノックなしの目に見えない SSH
- 使い捨てノック
- 短いアクセスウィンドウ
- 機能的NAT
- 永続的なファイアウォール
- 最小限の再帰 DNS (PDC が入るまで)

---

🎯 以上です!

👉 チリ_REF_0_チリ
👉 チリ_REF_0_チリ
