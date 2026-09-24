# 04. 携帯電話の Bluetooth 収容

発信用の SIM を挿した携帯電話（iPhone）を Bluetooth で Asterisk に接続し、`chan_mobile` を使って **離れた場所からその携帯回線で発信できる** ようにする。

> 元記事：[Asteriskで発信用の携帯電話をBluetoothで収容する](https://qiita.com/kmorimoto/items/3390c7c6565fcf249dfd)（Qiita, 2023-02-23）

## 背景

携帯キャリア（MNO・MVNO）のプランは多様になり、通話に使いたい回線とデータ通信に使いたい回線が一致しないことがある。通話回線だけを見ても、かけ放題契約の有無で使い分けることもある。デュアル SIM 対応のスマートフォンを使う方法もあるが、対応機種が必要なうえ、使い分けられるのは2回線までである。

この構成の発端は、私用の回線に加えて仕事用の回線が増えたことである。使っている iPhone SE（第1世代）は SIM を2枚挿せないため、2台を持ち歩くことになり面倒だった。

- **着信**：同一キャリアのファミリー契約にすれば、回線間の転送が無料でできる。
- **SMS**：iMessage を使えば、SIM を挿していない端末からも送受信できる。
- **発信**：その SIM を挿した端末から発信するしかない。← これを Asterisk で解決する

スマートフォンを買い替えれば済む話ではあるが、今後さらに回線が増えた場合や、格安のデータ専用回線だけを持ち歩く運用を考えると、遠隔で携帯回線から発信できる価値は小さくない。

## 方針

- 2回線のうち、回線 A の端末を持ち歩き、回線 B の端末は自宅に置く。
- 自宅の回線 B の端末と Raspberry Pi（Asterisk）を Bluetooth で接続する。
- Asterisk に `chan_mobile` を組み込み、回線 B の端末から発信できるようにする。

Asterisk に携帯回線を収容する方法として、個人で現実的なのは次の2つである。

| 方法 | 概要 | 評価 |
|---|---|---|
| **chan_mobile**（旧称 chan_cellphone） | スマートフォンを Bluetooth で収容 | 端末が安価で手に入り、長く使える → **採用** |
| chan_dongle | SIM を挿した USB モデムを直接収容 | 通話品質では有利。ただし対応モデムが入手しにくく、入手できても数年後のキャリアの停波には抗えない |

## 構成

```
(発信は回線B, 着信はキャリア内転送で回線Aへ)
     |                 (回線0ABJで発着信)
     |                         |                  (回線Aで発着信(+回線Bの転送着信))
     |                         |                                 |
+---------+               +--------+                        +---------+
|iPhone 5s|               |Asterisk|                        |iPhone SE|
|  SIM B  |--(Bluetooth)--|  0ABJ  |----------(4G)----------|  SIM A  |
+---------+               +--------+                        +---------+
   (自宅)                    (自宅)                            (自宅外)
                                                            Asterisk経由で
                                                            回線0ABJでの発着信と
                                                            回線Bでの発信が可能
```

- 持ち歩くのは右の iPhone SE（回線 A）だけ。デュアル SIM 対応機種には買い替えない。
- 左の iPhone 5s（回線 B）と Asterisk は自宅に置く。Asterisk は [03. ひかり電話への直収](03-hikari-denwa.md) で構築済みのもの。
- 回線 A と B は同一キャリアのファミリー契約で、回線間の転送通話料は無料。
  - 回線 B への着信は、Asterisk を経由せずキャリアの転送で回線 A に送っている。理由は後述の[通話品質](#通話品質について)のとおり。
  - キャリアが異なる場合でも、有料の転送サービスを使うか、回線 B の着信を Asterisk で受けて回線 A へ転送すれば同じことができる。

## 手順

作業は大きく2つに分かれる。

1. Raspberry Pi と iPhone を Bluetooth で接続する
2. Asterisk で Bluetooth 接続した iPhone を使えるようにする

2 は 1 ができていないと進められないため、この順で行う。

### 1. Raspberry Pi と iPhone を Bluetooth で接続する

#### Bluetooth ドングルを用意する

Raspberry Pi 内蔵の Bluetooth は、安定性や相性の面で評判がよくない。そのため USB の Bluetooth ドングルを別に用意した。多くの事例で動作実績のある **Buffalo BSBT4D09BK** を使っている。

#### BlueZ を入れる

Bluetooth を扱うには `bluetooth` サービスとプロトコルスタック（BlueZ）が必要である。Asterisk の `chan_mobile` をビルドするには開発用ヘッダも必要なので、あわせて入れる。

```sh
apt install bluez libbluetooth-dev
```

> [!IMPORTANT]
> BlueZ は最初から入っていることが多く `bluetoothctl` も使えるため、`libbluetooth-dev` を入れ忘れやすい。これがないと後で `chan_mobile` を組み込めない。

#### A2DP プロファイルを使えるようにする

この状態ではペアリングはできても接続（connect）できない。`service bluetooth status` を見ると、次のようなエラーが出ている。

```
bluetoothd[26285]: a2dp-source profile connect failed for xx:xx:xx:xx:xx:xx: Protocol not available
```

A2DP プロファイルが必要である。Debian で A2DP に対応する方法はいくつかあるが、手早いのは PulseAudio を入れることである。

```sh
apt install pulseaudio pulseaudio-module-bluetooth
```

PulseAudio をシステムデーモンとして動かすユニット定義ファイルを作る。

[`config/systemd/pulseaudio.service`](../config/systemd/pulseaudio.service) → `/etc/systemd/system/pulseaudio.service`

```ini
[Unit]
Description=Pulse Audio

[Service]
Type=simple
ExecStart=/usr/bin/pulseaudio --system --disallow-exit --disable-shm

[Install]
WantedBy=multi-user.target
```

PulseAudio から BlueZ へアクセスできるよう、D-Bus の設定に `org.bluez` を許可する行を追加する。

[`config/dbus/pulseaudio-system.conf`](../config/dbus/pulseaudio-system.conf) → `/etc/dbus-1/system.d/pulseaudio-system.conf`

```xml
<busconfig>

  <policy user="pulse">
    <allow own="org.pulseaudio.Server"/>
    <allow send_destination="org.bluez"/>
  </policy>

</busconfig>
```

起動時に Bluetooth 用モジュールを読み込むよう、`/etc/pulse/system.pa` に追記する。

[`config/pulse/system.pa.append`](../config/pulse/system.pa.append) の内容を `/etc/pulse/system.pa` に追記する。

```
### Automatically load driver modules for Bluetooth hardware
.ifexists module-bluetooth-policy.so
load-module module-bluetooth-policy
.endif

.ifexists module-bluetooth-discover.so
load-module module-bluetooth-discover
.endif
```

有効化して、念のため再起動する。

```sh
systemctl daemon-reload
systemctl enable pulseaudio
reboot
```

#### bluetooth サービスのエラーを抑える

環境によっては、`bluetooth` サービスの起動時に `Operation not permitted (1)` というエラーが出る。bluetoothd の起動オプションで SAP（SIM Access Profile）プラグインを無効にすると解消した。

`/etc/systemd/system/bluetooth.target.wants/bluetooth.service`

```ini
# 変更前
# ExecStart=/usr/lib/bluetooth/bluetoothd
# 変更後
ExecStart=/usr/lib/bluetooth/bluetoothd --noplugin=sap
```

#### ペアリングと接続の確認

ドングルが認識されていることを `hciconfig` で確認する。追加のドライバなしで認識された。

```
# hciconfig
hci0:   Type: Primary  Bus: USB
        BD Address: xx:xx:xx:xx:xx:xx  ACL MTU: 310:10  SCO MTU: 64:8
        UP RUNNING
        (略)
```

`bluetoothctl` で周囲をスキャンし、iPhone を見つける。

```
# bluetoothctl
Agent registered
[bluetooth]# power on
Changing power on succeeded
[bluetooth]# scan on
(略)
[bluetooth]# scan off
[bluetooth]# devices
Device xx:xx:xx:xx:xx:xx (周辺のデバイス)
(略)
Device xx:xx:xx:xx:xx:xx iPhone5s
```

表示されたアドレスが iPhone 側で表示されるアドレスと一致することを確かめてから、ペアリングする。iPhone 側でもパスキーの確認などの操作が必要なので、端末を手元に置いて行う。

```
[bluetooth]# pair xx:xx:xx:xx:xx:xx
Attempting to pair with xx:xx:xx:xx:xx:xx
[CHG] Device xx:xx:xx:xx:xx:xx Connected: yes
Request confirmation
[agent] Confirm passkey XXXXXX (yes/no): yes
(略)
[bluetooth]# trust xx:xx:xx:xx:xx:xx
[CHG] Device xx:xx:xx:xx:xx:xx Trusted: yes
Changing xx:xx:xx:xx:xx:xx trust succeeded
```

ペアリングはできたのに接続できない場合は、`systemctl status bluetooth` でエラーを確認する。正常に接続できていれば、`info` で `Connected: yes` と、`Handsfree Audio Gateway` などのプロファイルが表示される。

```
[iPhone5s]# info xx:xx:xx:xx:xx:xx
Device xx:xx:xx:xx:xx:xx (public)
        Name: iPhone5s
        Icon: phone
        Paired: yes
        Trusted: yes
        Blocked: no
        Connected: yes
        UUID: Audio Source              (...)
        UUID: Handsfree Audio Gateway   (...)
        (略)
```

`exit` で `bluetoothctl` を抜ける。

#### 自動再接続を設定する

このままでは、再起動などで切れた接続が元に戻らない。[noraworld/bluetoothctl-autoconnector](https://github.com/noraworld/bluetoothctl-autoconnector) を使い、cron で定期的に再接続させる。

```sh
cd /usr/local/bin/
git clone https://github.com/noraworld/bluetoothctl-autoconnector.git
cd bluetoothctl-autoconnector
./setup.sh
```

`setup.sh` を実行すると、次の cron が登録される。

```
*/1 * * * * /usr/local/bin/bluetoothctl-autoconnector/cron.sh
```

### 2. Asterisk に chan_mobile を組み込む

[03](03-hikari-denwa.md#2-asterisk-をソースからインストールする) のとおり Asterisk はソースから入れている。`chan_mobile` は既定では組み込まれないため、`--with-bluetooth` を付けて `configure` からやり直す。

```sh
cd /usr/src/asterisk-18.2.0/
./configure --with-bluetooth
make menuselect
```

`make menuselect` の画面で `Add-ons` を選び、`chan_mobile` を選択状態にする。

```
         --- Extended ---
     [*] chan_mobile
     [ ] chan_ooh323
     [ ] format_mp3
     XXX res_config_mysql
```

`chan_mobile` を選べない場合は、次のどちらかができていないことが多い。

1. `apt install libbluetooth-dev`
2. `./configure --with-bluetooth`

ビルドしてインストールする。

```sh
make
make install
```

`/usr/lib/asterisk/modules/chan_mobile.so` ができていれば成功である。現在の既定の `modules.conf` は `autoload=yes` なので、`modules.conf` で個別に読み込みを指定する必要はない。

### 3. chan_mobile.conf を設定する

Bluetooth ドングルのアドレスを調べる。

```
# hcitool dev
Devices:
        hci0    xx:xx:xx:xx:xx:xx
```

このアドレスを `/etc/asterisk/chan_mobile.conf` の `[adapter]` に設定して Asterisk を起動し、CLI の `mobile search` で iPhone の RFCOMM ポート番号を調べる。

```
pbx*CLI> mobile search
Address           Name                           Usable Type    Port
xx:xx:xx:xx:xx:xx iPhone5s                       Yes    Phone   8
```

> [!NOTE]
> iPhone が Bluetooth 接続中だと表示されない、近くで何度か Bluetooth をオン・オフしないと表示されないなど、表示されるまでに何度か試す必要があった。

ポート番号を含めて、端末の設定を追記する。

[`config/asterisk/chan_mobile.conf`](../config/asterisk/chan_mobile.conf)

```ini
[adapter]
id=hci0
address=xx:xx:xx:xx:xx:xx

[iPhone5s]
address=xx:xx:xx:xx:xx:xx       ; the address of the phone
port=8                          ; the rfcomm port number (from mobile search)
context=from-iphone5s           ; dialplan context for incoming calls
adapter=hci0                    ; adapter to use
;group=1                        ; this phone is in channel group 1
sms=no                          ; support SMS, defaults to yes
;nocallsetup=yes                ; set this only if your phone reports that it supports call progress notification, but does not do it. Motorola L6 for example.
```

セクション名（`[iPhone5s]`）と `context`（`from-iphone5s`）は、この後の `extensions.conf` で使う。Asterisk を再起動すると、`mobile show devices` で接続状態を確認できる。

```
pbx*CLI> mobile show devices
ID              Address           Group Adapter         Connected State      SMS
iPhone5s        xx:xx:xx:xx:xx:xx 0     hci0            Yes       Free       No
```

### 4. extensions.conf を設定する

発信のために `pjsip.conf` でトランクを作る必要はない。`chan_mobile.conf` の設定により、`Mobile/iPhone5s` というチャネルがそのまま使える。

すべての発信を回線 B から出すだけなら、次のように書けばよい。

```ini
exten => _X.,1,Dial(Mobile/iPhone5s/${EXTEN})
exten => _X.,n,Hangup
```

ひかり電話（0ABJ）と共存させる場合は、**先頭に 0 を1つ付けた番号（00 で始まる番号）を回線 B から発信** し、それ以外はひかり電話から発信するように振り分ける。

[`config/asterisk/extensions.conf`](../config/asterisk/extensions.conf) の `[default]`

```ini
exten => _00Z.,1,Dial(Mobile/iPhone5s/${EXTEN:1})
exten => _00Z.,n,Hangup
exten => _X.,1,Set(CALLERID(num)=${MYNUMBER})
exten => _X.,2,Set(CALLERID(name)=${MYNUMBER})
exten => _X.,3,Dial(PJSIP/${EXTEN}@HIKARI-DENWA)
exten => _X.,n,Hangup
```

- `0090-...`、`003-...` のように 00 で始まる番号は、先頭の 0 を取り除いて（`${EXTEN:1}`）回線 B から発信する。
- `090-...`、`03-...` などそれ以外の番号は、ひかり電話から発信する。
- この振り分けでは、ひかり電話から 00 で始まる番号へは発信できなくなる。そうした番号を使う場面はないと判断し、考慮していない。

回線 B への着信も Asterisk で処理したい場合は、`chan_mobile.conf` で指定した `from-iphone5s` のコンテキストを `extensions.conf` に作ればよい。

## 通話品質について

これで2台持ちから解放され、普段は回線 A の端末だけを持ち歩きつつ、必要なときに回線 B で発信できるようになった。

音質は非常にクリアで、SIP や Bluetooth を経由しているとは分からないほどである。ただし **遅延がはっきり分かる程度に発生する**。音質がよいために当初は遅延に気づかず、次のような会話になっていた。

```
自分「もしもし」
相手「....」
自分「もしもし聞こえますか！」
相手「もしもし」
自分「先日の件ですが～」
相手「ああ、聞こえます」
自分「え？聞こえにくいですか？」
相手「あ、はい」
自分「え、やっぱり聞こえない!?」
相手「いや、聞こえますよ」
自分「どっちなんですか!!」
```

原因は切り分けていないが、経験的には Bluetooth 区間の遅延と考えている。A2DP の SBC コーデックの影響かもしれず、aptX 対応にする、ドングルを替えるといった対策が考えられるが、未調査である。現状は、遅延を前提に相手の発言に少しかぶせ気味に話すことで対処している。

このため、回線 B への着信は Asterisk を経由せず、回線 A へのキャリア内転送にしている。将来、データ専用回線だけを持ち歩く運用に移り、すべての発着信を Asterisk 経由にする場合は、原因の究明と対策が必要になる。
