# NSD構築およびBINDからの移行 手順書（systemctl対応版）

## 1. 作業概要
- 目的：既存BIND(named)からNSDへ権威DNSを移行し、指定ゾーンを応答可能にする
- 影響：切替（BIND停止〜NSD起動）時にDNS応答が一時停止（目安1〜2分）
- 対象：Rocky Linux 9 / AlmaLinux 9（dnf環境）
- 備考：NSDは「権威DNS専用」。再帰DNSにはならない

---

## 2. パラメータ（環境に合わせて置き換え）
例：
- ZONE_NAME：teamb2.entrycl.net
- ZONE_FILE：teamb2.entrycl.net.zone
- ZONES_DIR：/etc/nsd
- NSD_VERSION：4.14.1（例。任意の安定版を固定する）
- NSD_LISTEN：0.0.0.0:53

---

## 3. 事前準備（証跡と状態確認）

### 3.1 証跡取得（任意）
```bash
sudo script /var/tmp/nsd_migration_$(date +%Y%m%d_%H%M%S).log
```
### 3.2 ディスク・ポート確認
```bash
df -h /
sudo ss -lnpt | grep :53 || true
sudo ss -lnup | grep :53 || true
期待値（移行前）：named が :53 を使用中
```

## 4. NSDインストール（ソースビルド）

Rocky/Almaでも、バージョン固定・再現性を重視するならソースビルドが安全です。
※Amazon Linux 2023ではdnfで入らないケースが多く、ソースビルド推奨。

### 4.1 依存パッケージ導入
```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y openssl-devel libevent-devel wget tar
```

### 4.2 ソース取得・展開
```bash
sudo mkdir -p /usr/local/src
cd /usr/local/src

sudo wget https://www.nlnetlabs.nl/downloads/nsd/nsd-${NSD_VERSION}.tar.gz
sudo tar -xzf nsd-${NSD_VERSION}.tar.gz
cd nsd-${NSD_VERSION}
```

### 4.3 configure / make / install
標準（Rocky/Almaで多い）
```bash
sudo ./configure --prefix=/usr/local --sysconfdir=/etc --localstatedir=/var
sudo make -j"$(nproc)"
sudo make install
```
### dnstapエラーが出る場合（Amazon Linux 2023等）

protoc / protoc-c が無い と言われたら、dnstapを無効化します：
```bash
sudo ./configure --prefix=/usr/local --sysconfdir=/etc --localstatedir=/var --disable-dnstap
sudo make -j"$(nproc)"
sudo make install
```
### 4.4 インストール確認
```bash
/usr/local/sbin/nsd -v
```
## 5. NSD実行ユーザーとディレクトリ
```bash
sudo useradd -r -s /sbin/nologin nsd 2>/dev/null || true

sudo mkdir -p /etc/nsd
sudo mkdir -p /var/db/nsd
sudo mkdir -p /run/nsd
sudo touch /var/log/nsd.log

sudo chown -R nsd:nsd /var/db/nsd /run/nsd /var/log/nsd.log
sudo chmod 755 /etc/nsd
```

## 6. nsd-control用証明書（任意だが推奨）

nsd-control で reload/reconfig ができるようになります。
```bash
sudo /usr/local/sbin/nsd-control-setup
ls -l /etc/nsd | grep -E "pem|key" || true
```

## 7. NSD設定ファイル（/etc/nsd/nsd.conf）作成（vi版）
```bash
sudo vi /etc/nsd/nsd.conf
```

## 7. NSD設定ファイル（/etc/nsd/nsd.conf）作成（vi版）
```bash
sudo vi /etc/nsd/nsd.conf
```
以下を貼り付けて保存（ESC → :wq）：
```bash
server:
  ip-address: 0.0.0.0
  do-ip4: yes
  username: nsd
  zonesdir: "/etc/nsd"
  logfile: "/var/log/nsd.log"
  pidfile: "/run/nsd/nsd.pid"
  database: "/var/db/nsd/nsd.db"
  port: 53

remote-control:
  control-enable: yes
  control-interface: 127.0.0.1
  server-key-file: "/etc/nsd/nsd_server.key"
  server-cert-file: "/etc/nsd/nsd_server.pem"
  control-key-file: "/etc/nsd/nsd_control.key"
  control-cert-file: "/etc/nsd/nsd_control.pem"

zone:
  name: "teamb2.entrycl.net"
  zonefile: "teamb2.entrycl.net.zone"
```

### 7.1 設定チェック
```bash
sudo /usr/local/sbin/nsd-checkconf /etc/nsd/nsd.conf
```

## 8. ゾーンファイル作成（/etc/nsd/teamb2.entrycl.net.zone）（vi版）
```bash
sudo vi /etc/nsd/teamb2.entrycl.net.zone
```

例（※値は環境に合わせて）：

```bash
$ORIGIN teamb2.entrycl.net.
$TTL 3600
@ IN SOA ns.teamb2.entrycl.net. root.teamb2.entrycl.net. (
  2026030604 ; serial
  3600
  900
  604800
  300 )

@  IN NS ns.teamb2.entrycl.net.
ns IN A  <NSD_EIP_OR_PUBLIC_IP>

; ALBを使う場合（推奨）：CNAMEでALB DNS名へ
www IN CNAME CL-TeamE-test0-2044806691.us-west-2.elb.amazonaws.com.
```
### 8.1 ゾーン構文チェック
```bash
sudo /usr/local/sbin/nsd-checkzone teamb2.entrycl.net /etc/nsd/teamb2.entrycl.net.zone
```

## 9. systemctlで管理できるようにする（nsd.service作成）
### 9.1 unitファイル作成
```bash
sudo vi /etc/systemd/system/nsd.service
```

貼り付け（ESC → :wq）：
```bash
[Unit]
Description=NSD DNS Server
After=network.target

[Service]
Type=forking
User=nsd
ExecStart=/usr/local/sbin/nsd -c /etc/nsd/nsd.conf
PIDFile=/run/nsd/nsd.pid
Restart=on-failure
RestartSec=2s

[Install]
WantedBy=multi-user.target
```
### 9.2 systemd反映・起動
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nsd
sudo systemctl status nsd --no-pager
```
## 10. BIND → NSD 切替（ポート53競合防止）

#### namedが生きているとNSDが正しく応答しない

### 10.1 BIND停止・無効化
```bash
sudo systemctl stop named
sudo systemctl disable named
# 念のため（任意）
sudo systemctl mask named
```
### 10.2 NSD再起動（systemctl）
```bash
sudo systemctl restart nsd
sudo systemctl status nsd --no-pager
```
## 11. 動作確認
### 11.1 待受確認（TCP/UDP両方）
```bash
sudo ss -lnpt | grep :53
sudo ss -lnup | grep :53
```
### 11.2 ローカルdig確認（権威応答 aa が付く）
```bash
dig @127.0.0.1 teamb2.entrycl.net SOA +norecurse
dig @127.0.0.1 teamb2.entrycl.net NS  +norecurse
dig @127.0.0.1 www.teamb2.entrycl.net +norecurse
``` 
## 12. 反映（ゾーン更新時）

ゾーンファイルを編集したら、必ず serial を増やしてから以下。

#### nsd-controlが使える場合（推奨）
```bash
sudo /usr/local/sbin/nsd-control reconfig
sudo /usr/local/sbin/nsd-control reload teamb2.entrycl.net
```
#### 使えない場合
```bash
sudo systemctl restart nsd
```

## 13. 切り戻し手順（NSDが動かない場合）
```bash
sudo systemctl stop nsd
sudo systemctl unmask named 2>/dev/null || true
sudo systemctl enable --now named
sudo systemctl status named --no-pager
sudo ss -lnpt | grep :53
```

## 14. よくある詰まりポイント

#### namedが起動したまま → REFUSED / 応答不正 → named停止が必須

### zonesdir/zonefileのパス違い → NSDがゾーンを読めない → nsd-checkconf と nsd-checkzone で事前検知

### CNAME末尾の . 忘れ → elb.amazonaws.com.<origin> になって壊れる → 末尾ドット必須

### ALBをAで固定してしまう → IPが変わり壊れる → ALBはCNAME


---
