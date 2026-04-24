# 🧩 VOID LINUX 튜토리얼 — 보안 체계 구현 – VOIDBR LABORATORY

🔥 방화벽 com Void Linux, IPTables, NAT, Port Knocking.

---

# 🔥 방화벽 + 프록시가 SAMBA4 도메인(VOIDBR.NET)에 통합되었습니다.

## 🎯 이 튜토리얼의 목표는 **Debian 13**에서 **방화벽 + Squid 프록시** 서버를 구성하고 **주 게이트웨이(192.168.70.254)** 역할을 하며 **VOIDBR.NET** 도메인(도메인 컨트롤러: **192.168.70.253**)에 통합하여 **Active Directory(NTLM)**를 통해 Squid에서 사용자를 인증하고 **차단 정책**을 적용하는 것입니다.

---

## 🌐 1. 네트워크 토폴로지 - 기능, IP 주소 지정 및 이름:

- 도메인: VOIDBR.NET

- 방화벽/프록시: SRVFIREWALL 192.168.70.254

- 도메인 컨트롤러: SRVDC01 192.168.70.253

- 파일 서버: SRVFILES 192.168.70.252

- WAN: `eth0` → 192.168.122.254/24(게이트웨이: 192.168.122.1)

- LAN: `eth1` → 192.168.70.254/24

---

## 😀 배포 대상

### 입력

-> LAN/DMZ에서 방화벽으로의 ping 허용
-> LAN/DMZ에서 SSH를 방화벽으로 연결하도록 허용
-> LAN을 통한 프록시 액세스 허용
-> 포트 노크가 있는 RECENT 모듈을 통해 외부 SSH를 방화벽에 허용

### 앞으로

-> LAN 액세스 AD 허용 -> DNS, SMB, DHCP, NETBIOS)
-> DMZ의 인터넷 접속을 허용하세요
-> AD가 인터넷의 DNS에 액세스하도록 허용
-> LAN은 인터넷의 이메일에 액세스해야 합니다.
-> LAN에서 DMZ 핑 허용
-> LAN은 수익망, 정부 등에 액세스해야 합니다.

### NAT

-> 웹 서버 게시 -> 80/443
-> LAN 및 DMZ용 아웃바운드 NAT

---

## ✅ 2. 네트워크 다이어그램

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

다른 각도에서 보기

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

방화벽은 인터넷에 노출되는 유일한 호스트입니다.

## ✅ 3. 목표 및 가정

- 기본 정책 거부
- 활성 IPv4 라우팅
- 스캐너가 문을 보지 못함
- 유일한 진입점인 방화벽
- 게시된 웹 대시보드가 없습니다.
- 포트 노킹으로 보호되는 SSH
- Fail2ban을 통한 무차별 대입 제어
- LAN용 제어 NAT
- SSH 터널을 통한 원격 관리

## 참고: 루트 사용자로 실행되는 튜토리얼!!

## ✅ 4. 필요한 패키지 업데이트 및 설치

시스템 업데이트

```bash
vinstall -Syu
```

패키지 설치

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

## dnsmasq 파일의 기본 구성

```bash
vim /etc/dnsmasq.conf
```

콘텐츠:

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

## DNSMASQ 활성화

```bash
vservice enable dnsmasq
```

## 온라인 서비스 검증

```bash
vsv
```

## ✅ 5. SSH 구성

```bash
vim /etc/ssh/sshd_config.d/10-custom.conf
```

뾰족한 선을 추가하세요.

```bash
Port 22254
PermitRootLogin yes
SyslogFacility AUTH
LogLevel INFO
```

## 서비스 활성화

```bash
vservice enable sshd
```

## 전체 배포 후:

- 루트 로그인 비활성화

- 키 인증만 사용

## ✅ 6. 방화벽 네트워크 설정

```bash
vim /etc/dhcpcd.conf
```

콘텐츠

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

적용하다

```bash
vservice restart dhcpcd
```

## ✅ 7. 포트 노킹 – 커널 지원

필요한 모듈을 로드하세요.

```bash
modprobe xt_recent
```

검증:

```bash
lsmod | grep xt_recent
```

예상되는 결과

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## 파일을 생성하여 이러한 모듈을 지속적으로 추가하십시오.

```bash
echo 'xt_recent' >/etc/modules-load.d/xt_recent.conf
```

## ✅ 8. 방화벽 IP테이블

방화벽 네트워크 카드 간 라우팅 활성화

```bash
echo 'net.ipv4.ip_forward=1' >/etc/sysctl.conf
```

재부팅 없이 적용:

```bash
sysctl --system
```

모든 사전 규칙을 지웁니다(있는 경우).

```bash
iptables -F
iptables -t nat -F
iptables -X
```

🔐 FILTER 테이블 규칙

기본 정책을 설정합니다.

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

입력(방화벽 액세스)

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

🔐 FORWARD 테이블 규칙(네트워크 간 트래픽)

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

🔐 NAT 테이블 규칙

```bash
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o eth0 -j MASQUERADE
```

🧾 손실된 패킷 로그(선택 사항)

```bash
iptables -A INPUT -j LOG -m limit --limit 5/min --log-prefix "DROP_INPUT: "
iptables -A FORWARD -j LOG -m limit --limit 5/min --log-prefix "DROP_FORWARD: "
```

# 노크: IP를 기록합니다.

iptables -A INPUT -i eth0 -p tcp --dport 34567 -m conntrack --ctstate NEW -m 최근 --set --name KNOCK --rsource -j DROP

# SSH가 한 번 출시되고 노크가 제거됩니다.

iptables -A INPUT -m conntrack --ctstate 설정됨, 관련됨 -j 수락
iptables -A INPUT -i eth0 -p tcp --dport 22254 -m conntrack --ctstate NEW -m 최근 --rcheck --seconds 15 --name KNOCK --rsource -m 최근 --remove --name KNOCK --rsource -j ACCEPT

규칙을 지속적으로 저장합니다.

```bash
iptables-save > /etc/iptables/iptables.rules
```

생성된 규칙을 나열합니다(번호, 자세한 정보, 목록).

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

## ✅ 9. 포트 노킹 테스트 및 검증(핫)

방화벽 없이 터미널 노크를 모니터링하세요.

```bash
tcpdump -ni eth0 tcp port 34567
```

외부 액세스를 통해 고객의 노크 보내기

```bash
nc -z 192.168.122.254 12345
```

✔ SYN 도착
✔ 삭제되었습니다.
✔ 등록 상태 유지
✔ 상태가 표시됩니다

tcpdump의 예상 결과

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

중요한 기술 노트

- RST는 TCP 스택을 통해 전송됩니다.
- 패키지가 xt_recent에 의해 등록되었습니다.
- 포트가 서비스로 응답하지 않습니다.
- 배너나 지문이 없습니다.

IP 등록 유효성 검사(15초 이내에 삭제됩니다)

```bash
cat /proc/net/xt_recent/KNOCK
```

예상되는 결과

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

## ✅ 10. 외부 SSH 유효성 검사 수행

노크를 실행

```bash
nc -z 192.168.122.254 34567
```

15초 이내에 접속

```bash
ssh -p 22254 anon@192.168.122.254
```

권장 별칭

```bash
vim ~/.bashrc
```

콘텐츠

```bash
alias knock='nc -z 192.168.122.254 34567'
alias firewall='ssh -p 22254 anon@192.168.122.254'
```

유효성 검사를 위해 파일을 다시 읽습니다.

```bash
source ~/.bashrc
```

## 11. 🎉 체크리스트 최종

- 노크 없이 보이지 않는 SSH
- 일회용 노크
- 짧은 액세스 창
- 노크 무시 금지
- 기능적 NAT
- 영구 방화벽
- 최소 재귀 DNS(PDC가 들어갈 때까지)

---

🎯 그게 전부입니다!

👉 https://t.me/m3vorak
👉 https://t.me/vcatafesta
