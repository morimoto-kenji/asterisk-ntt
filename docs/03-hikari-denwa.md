# 03. ひかり電話への直収

フレッツ光ネクスト（NGN）のひかり電話へ、ONU 配下に置いた Asterisk を **pjsip（`res_pjsip`）で直接接続** する。

ひかり電話へ Asterisk を直収する情報は以前からあるが、数が少なく断片的で、古いものが多い。また Asterisk は `chan_sip` から `res_pjsip` へ移行しており、`sip.conf` 前提の従来の情報はそのままでは使えない。そこで pjsip での直収を、情報取得から発着信まで通しでまとめる。あわせて、今後の仕様変更に対応する手がかりとなるよう、試行錯誤の結果も残す。

> 元記事：[NTT光ネクストのひかり電話へAsteriskを直接接続する](https://qiita.com/kmorimoto/items/d99cd9edcf7436eea7cc)（Qiita, 2021-02-19）

## 方針

1. フレッツ網から、ひかり電話に必要な情報を DHCP で取得する
2. Asterisk をソースからインストールする
3. 取得した情報をもとに `pjsip.conf` を設定し、登録（REGISTER）と発着信を行う

## 前提

- ネットワーク構成は [README の全体構成](../README.md#全体構成) のとおり。Raspberry Pi の eth0 を ONU 配下に直接接続している。
- Raspberry Pi でも IPv4 の PPPoE 接続をしている（[02. Route53 による DDNS](02-route53-ddns.md)）。
- DHCP クライアントを dhclient に切り替えてある（[01. dhclient への切り替え](01-dhclient.md)）。
- OS は Raspbian 10 (buster)。

## 手順

### 1. フレッツ網から DHCP で情報を取得する

SIP サーバの IP アドレスや電話番号などは、フレッツ網から DHCP オプション（[RFC 3361](https://tools.ietf.org/html/rfc3361) など）で取得する。かつては PPPoE 接続して POST で情報を得ていたようだが、現在は不要（あるいは不可）である。B フレッツや光プレミアムの時代の方式と思われる。

dhclient の設定は [voip-info.jp](http://www.voip-info.jp/index.php/%E5%88%A9%E7%94%A8%E8%80%85:Pin_ptr#DHCP) で公開されているものを使った。

[`config/dhcp/dhclient.conf`](../config/dhcp/dhclient.conf) の内容を `/etc/dhcp/dhclient.conf` に追記する。

```
option ip-sip-servers code 120 = { boolean, array of ip-address };
option vendor-class.ntt code 210 = string;
option space ntt code width 1 length width 1 hash size 7;
option ntt.mac code 201 = string;
option ntt.number code 202 = text;
option ntt.domain code 204 = domain-list;
option ntt.firmware code 210 = domain-list;
option vendor.ntt code 210 = encapsulate ntt;

interface "eth0" {
  send dhcp-client-identifier = hardware;
  request subnet-mask, routers, ip-sip-servers, rfc3442-classless-static-routes, vivso;
  send vendor-class.ntt = concat(06, suffix(hardware, 6));
}
```

DHCPv6 でも情報は取れるが、直収には必須でないため使っていない。この `dhclient.conf` でほぼ必要十分である。

西日本で取得できた内容は以下のとおり（`/var/lib/dhcp/dhclient.leases`）。東日本でも、IP アドレスとドメイン名以外は同じ項目が取れると思われる。

```
lease {
  interface "eth0";
  fixed-address 124.xxx.xxx.xxx;
  option subnet-mask 255.255.255.252;
  option dhcp-lease-time 14400;
  option routers 124.xxx.xxx.xxx;
  option dhcp-message-type 5;
  option dhcp-server-identifier 124.xxx.xxx.xxx;
  option dhcp-renewal-time 7200;
  option ip-sip-servers true 124.xxx.xxx.1;
  option dhcp-rebinding-time 10800;
  option rfc3442-classless-static-routes xxx,xxx,xxx,....;
  option vivso xx:xx:xx:....;
  option vendor.ntt xx:xx:xx:....;
  option ntt.domain "ntt-west.ne.jp.";
  option ntt.firmware "www.verinfo.hgw.flets-west.jp.";
  option ntt.mac xx:xx:xx:xx:xx:xx;
  option ntt.number "0xxxxxxxxx";
  renew 3 2021/02/xx xx:xx:xx;
  rebind 3 2021/02/xx xx:xx:xx;
  expire 3 2021/02/xx xx:xx:xx;
}
```

以降の設定で使うのは次の3つである。

| DHCP オプション | 意味 | 使う場所 |
|---|---|---|
| `ip-sip-servers` | SIP サーバの IP アドレス | `pjsip.conf` の `contact` / `match`、`/etc/hosts` |
| `ntt.domain` | SIP ドメイン | `pjsip.conf` の `client_uri` / `server_uri` / `from_domain` |
| `ntt.number` | 電話番号 | `pjsip.conf` の `contact_user` / `client_uri`、`extensions.conf` |

#### DNS は /etc/hosts で代用する

DHCPv4 では DNS サーバのアドレスが取れない（DHCPv6 では取れる）。ひかり電話で名前解決が必要なのは `ntt.domain` → SIP サーバの1件だけなので、`/etc/hosts` で対応した。

```sh
cat <<'EOF' >> /etc/hosts
124.xxx.xxx.1 ntt-west.ne.jp
EOF
```

経路情報は DHCP（`rfc3442-classless-static-routes`）で自動的に設定されるため、手作業で追加する必要はない。

#### 確認

`ifconfig` と `route` の結果は次のようになる（値は環境により異なる）。

```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 124.xxx.xxx.xxx  netmask 255.255.255.252  broadcast 124.xxx.xxx.xxx
        (略)

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.xx  netmask 255.255.255.0  broadcast 192.168.0.255
        (略)

ppp0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1454
        inet 153.xxx.xxx.xxx  netmask 255.255.255.255  destination 153.xxx.xxx.xxx
        (略)
```

```
カーネルIP経路テーブル
受信先サイト    ゲートウェイ    ネットマスク   フラグ Metric Ref 使用数 インタフェース
default         0.0.0.0         0.0.0.0         U     0      0        0 ppp0
124.xxx.0.0     124.xxx.xxx.xxx 255.255.xxx.0   UG    0      0        0 eth0
124.xxx.xxx.xxx 0.0.0.0         255.255.255.252 U     0      0        0 eth0
124.xxx.xxx.0   124.xxx.xxx.xxx 255.255.xxx.0   UG    0      0        0 eth0
153.xxx.xxx.xxx 0.0.0.0         255.255.255.255 UH    0      0        0 ppp0
192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 eth1
xxx.xxx.xxx.0   124.xxx.xxx.xxx 255.255.224.0   UG    0      0        0 eth0
```

確認するポイントは次の2点である。

- eth0 に、フレッツ網から DHCP で取得したアドレス（124.xxx.xxx.xxx）が付いている
- NGN 向けの経路が eth0 を向いている（デフォルトルートは ppp0）

### 2. Asterisk をソースからインストールする

直収に必要な `pjsip.conf` のパラメータ `disable_rport` は、ChangeLog（[16](http://downloads.asterisk.org/pub/telephony/asterisk/ChangeLog-16-current)、[18](http://downloads.asterisk.org/pub/telephony/asterisk/ChangeLog-18-current)）によると、Asterisk 16.12.0 以降または 18.0.0 以降で使える。一方、2021年2月時点の Raspbian で apt から入る Asterisk は 16.2.1 で、このパラメータが使えない。そのため[公式サイト](https://www.asterisk.org/downloads/)からソースを取得してビルドする。

```sh
cd /usr/src
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-18.2.0.tar.gz
tar -zxvf asterisk-18.2.0.tar.gz
cd asterisk-18.2.0
./configure
```

#### 依存パッケージ

[公式ドキュメント](https://wiki.asterisk.org/wiki/display/AST/Checking+Asterisk+Requirements)にあるとおり、まずそのまま `./configure` を実行し、不足を指摘されたパッケージを都度 apt で入れた。必要なパッケージは Asterisk や OS のバージョンで変わるため、既存の記事のリストをそのまま使うのではなく、今回の組み合わせで実際に必要なものを確かめたかったためである。

Raspbian 10 (buster) ＋ Asterisk 18.2.0 で不足を指摘されたのは次のとおり。

```sh
apt install libedit-dev uuid uuid-runtime uuid-dev libjansson-dev libxml2-dev sqlite3 libsqlite3-dev
```

加えて、Asterisk が `/etc/hosts` を参照して名前解決するには `res_resolver_unbound` が必要なため（[公式コミュニティ](https://community.asterisk.org/t/pjsip-dns-resolution-with-etc-hosts-asterisk-12-4/70637)、[公式ドキュメント](https://www.asterisk.org/res_resolver_unbound/)）、以下も入れる。

```sh
apt install libunbound-dev
```

pjproject は、Asterisk 15.0.0 以降では既定で同梱・ビルドされる（[公式ドキュメント](https://wiki.asterisk.org/wiki/display/AST/PJSIP-pjproject)）ため、`configure` に引数は付けていない。成功すると最後に次のように表示される。

```
configure: Package configured for:
configure: OS type  : linux-gnueabihf
configure: Host CPU : armv7l
configure: build-cpu:vendor:os: armv7l : unknown : linux-gnueabihf :
configure: host-cpu:vendor:os: armv7l : unknown : linux-gnueabihf :
```

> [!TIP]
> 後で [04. 携帯電話の Bluetooth 収容](04-bluetooth-mobile.md) を行う場合は、`--with-bluetooth` を付けて `configure` し、`make menuselect` で `chan_mobile` を有効にする必要がある。

#### ビルドとインストール

[公式ドキュメント](https://wiki.asterisk.org/wiki/display/AST/Building+and+Installing+Asterisk)のとおりにビルドする。

```sh
make
make install
```

続いて、起動スクリプト（[Installing Initialization Scripts](https://wiki.asterisk.org/wiki/display/AST/Installing+Initialization+Scripts)）とサンプル設定（[Installing Sample Files](https://wiki.asterisk.org/wiki/display/AST/Installing+Sample+Files)）、logrotate の設定を入れる。

```sh
make config
make samples
make install-logrotate
systemctl daemon-reload
systemctl enable asterisk
```

これで systemd のユニット定義ファイル、`/etc/asterisk/` 以下のサンプル設定、`/etc/logrotate.d/asterisk` が作られる。

### 3. pjsip.conf を設定する

Asterisk には `sip.conf` を `pjsip.conf` に変換するスクリプト（`contrib/scripts/sip_to_pjsip/sip_to_pjsip.py`）がある。ただし[公式ドキュメント](https://wiki.asterisk.org/wiki/display/AST/Migrating+from+chan_sip+to+res_pjsip)にもあるとおり、常に互換性のある設定を出力できるわけではない。そこでスクリプトの出力をたたき台にして、登録と発着信ができるパラメータを探った。最終的に動いた設定が次のものである。

[`config/asterisk/pjsip.conf`](../config/asterisk/pjsip.conf)（ひかり電話部分を抜粋）

```ini
[global]
type = global
debug = no

[system]
type = system
disable_rport = yes ;Viaヘッダにrportを入れないために必須

[transport-udp]
type = transport
protocol = udp
bind = 0.0.0.0

[reg_HIKARI-DENWA]
type = registration
contact_user = 0xxxxxxxxx ;電話番号（DHCPのntt.number）
transport = transport-udp
client_uri = sip:0xxxxxxxxx@ntt-west.ne.jp ;"電話番号@ntt.domain" であること
server_uri = sip:ntt-west.ne.jp ;ntt.domain であること

[HIKARI-DENWA]
type = aor
contact = sip:124.xxx.xxx.1 ;DHCPのip-sip-servers

[HIKARI-DENWA]
type = identify
endpoint = HIKARI-DENWA
match = 124.xxx.xxx.1 ;DHCPのip-sip-servers

[HIKARI-DENWA]
type = endpoint
context = incoming ;extensions.confの着信用コンテキスト
disallow = all
allow = ulaw
rtp_symmetric = no ;Viaヘッダにrportを入れないために必須
force_rport = no ;同上
rewrite_contact = no ;同上
direct_media = no ;RTPをAsteriskで中継する
from_domain = ntt-west.ne.jp ;ntt.domain であること
aors = HIKARI-DENWA
```

#### 重要なポイント（試行錯誤で判明したこと）

- **`disable_rport = yes` は必須。**
  pjsip は既定で Via ヘッダに `rport` を付ける。`rport` が付いていると、登録と着信はできても **発信だけができない**（INVITE に対して `400 Bad Request` が返る）。
- **`client_uri` / `server_uri` は `ntt.domain`（ここでは `ntt-west.ne.jp`）にする。**
  `sip_to_pjsip.py` で変換すると IP アドレスになることがあるが、それでは何を送っても `400 Bad Request` が返る。
  - ドメインを指定しただけでは名前解決できず `No Response received` になる。そのため `/etc/hosts` に登録し、`libunbound-dev` を入れて Asterisk が `/etc/hosts` を参照できるようにしている。
- **`contact` / `match` は SIP サーバの IP アドレス（`ip-sip-servers`）でよい。**
- **`allow = ulaw`**：[公式ドキュメント](https://wiki.asterisk.org/wiki/display/AST/Asterisk+18+Configuration_res_pjsip)では既定値が空のため、NTT の技術参考資料（[東](https://flets.com/hikaridenwa/use/data.html)、[西](https://www.ntt-west.co.jp/info/gisanshi/)）を確認して G.711 μ-law を指定した。
- **`rtp_symmetric` / `force_rport` / `rewrite_contact` を `no` にするのは必須。**
  [公式ドキュメント](https://wiki.asterisk.org/wiki/display/AST/Migrating+from+chan_sip+to+res_pjsip)にあるとおり、`sip.conf` の `nat=never` に相当する設定で、`disable_rport` と合わせて Via ヘッダに `rport` が入らなくなる。
- **`direct_media = no`**：インターネット側の端末とひかり電話の SIP サーバが直接 RTP をやりとりできるはずはないため、Asterisk で中継する設定にした（必須かどうかは未検証）。

#### 設定しなかったパラメータ

- `dtmf_mode`：既定値が RFC 4733 で、NTT の技術参考資料でも RFC 4733 に対応しているため指定していない。`sip.conf` では `dtmfmode=rfc2833` とすることが多かったが、RFC 2833 を改訂したものが [RFC 4733](https://tools.ietf.org/html/rfc4733) である。
- `timers` / `timers_min_se` / `timers_sess_expires`：セッションタイマの設定。今のところ不要だった。意図せず通話が切れる、あるいは切れないといった症状が出たら調整する。
- `tos`：指定しなくても動いた。QoS の観点では指定すべきかもしれない。

### 4. extensions.conf を設定する

直収そのものとは関係ないが、参考としてダイヤルプランも載せる。

[`config/asterisk/extensions.conf`](../config/asterisk/extensions.conf)（ひかり電話部分を抜粋）

```ini
[general]
writeprotect=no
priorityjumping=no

[globals]
USEVOICEMAIL=NO
MYNUMBER=0xxxxxxxxx
PHONEALL=PJSIP/100

[default]
exten => 100,1,Dial(PJSIP/100)
exten => 100,n,Hangup
exten => _X.,1,Dial(PJSIP/${EXTEN}@HIKARI-DENWA)
exten => _X.,n,Hangup

[incoming]
exten => _X.,1,Set(free_dial="0120")
exten => _X.,2,Set(free_call="0800")
exten => _X.,3,GotoIf($["${CALLERID(num)}"="anonymous"]?99:4)
exten => _X.,4,GotoIf($[${CALLERID(num)}:${free_dial}]?99:5)
exten => _X.,5,GotoIf($[${CALLERID(num)}:${free_call}]?99:6)
exten => _X.,6,Dial(${PHONEALL})
exten => _X.,99,Answer()
exten => _X.,n,Hangup
```

- `[default]`：内線 100 を呼び出し、それ以外の番号はひかり電話から外線発信する。
- `[incoming]`：ひかり電話からの着信。非通知（`anonymous`）と、0120・0800 から始まる番号（フリーダイヤル）は、電話機を鳴らさずに応答して切断する。それ以外は `PHONEALL` の電話機を鳴らす。
  - フリーダイヤルを拒否する設定は、5ch.net でもらったヒントをもとにしている。

## 補足：DHCP の情報が変わった場合

電話番号や SIP サーバのアドレスは `pjsip.conf` / `extensions.conf` に直接書いているため、DHCP で配布される値が変わると追随できない。[voip-info.jp](http://www.voip-info.jp/index.php/%E5%88%A9%E7%94%A8%E8%80%85:Pin_ptr#ntt.conf) では dhclient の hook で `sip.conf` を書き換える方法が紹介されている。

ただ、`dhclient.leases` を観察している限りこれらの値が変わったことはなく、今後も変わらないと考えられるため、自動反映の仕組みは作っていない。
