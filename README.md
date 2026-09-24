# asterisk-ntt

Raspberry Pi 上の Asterisk を、NTT フレッツ光ネクスト（NGN）のひかり電話へ **ルータを介さず直接接続（直収）** し、自宅の IP-PBX として運用するための手順と参考設定です。

`chan_sip` ではなく、現行の **`res_pjsip`（pjsip.conf）で直収する構成** を、DHCP による情報取得から発着信まで通しでまとめています。あわせて、Bluetooth で接続した携帯電話を Asterisk の発信回線として収容する方法も収録しています。

> [!NOTE]
> 本リポジトリは、2021〜2023 年に Qiita へ投稿した記事を、GitHub 向けに再構成して再掲したものです。
> 元記事は末尾の「[元記事](#元記事)」を参照してください。

## 特徴

- **pjsip でのひかり電話直収**：`sip.conf` 前提の情報が多いなか、`pjsip.conf` で登録・発信・着信を実現した設定を掲載
- **ハマりどころの明示**：`disable_rport` がないと発信だけ `400 Bad Request` になる、など試行錯誤で判明したポイントを解説
- **周辺の準備も通しで**：DHCP オプションの取得（dhclient）、PPPoE 接続時の DNS 更新（Route53）まで含めて一式
- **携帯電話の収容**：`chan_mobile` で iPhone を Bluetooth 接続し、SIM を挿した端末を離れた場所から発信回線として利用

## 全体構成

```
            NTT/NGN
               |
             +---+
             |ONU|
             +---+
               |
             +---+
       +-----|HUB|--------+
       |     +---+        |
  (Public IP)        (Public IP)
   +------+        +-------------+                 +-----------+
   |      |        |    (eth0)   |                 |  iPhone   |
   |Router|        | Raspberry Pi|--(Bluetooth)----|  (SIM B)  |
   |      |        |  Asterisk   |                 +-----------+
   |      |        |    (eth1)   |
   +------+        +-------------+
 (192.168.0.x)      (192.168.0.x)
       |                  |
       |     +---+        |
       +-----|HUB|--------+
             +---+
              | |
       +------+ +---------+
       |                  |
     +---+           +---------+
     |PC |           |SIP Phone|
     +---+           +---------+
```

- Raspberry Pi のオンボード NIC（eth0）を ONU 配下に直接接続し、IPv4 PPPoE 接続とひかり電話用の DHCP を eth0 で行う
  - ルータ配下に置かないのは、NAT 越えの問題を避けるため
- LAN 側は USB NIC（eth1）を追加して接続。eth0〜eth1 間のルーティング／フォワーディングはしない
- 宅内の SIP 電話機は eth1 側から Asterisk に内線登録する

## ドキュメント

番号順に読むことを想定しています。中心は **03** で、01・02 はその前提、04 は発展編です。

| # | ドキュメント | 位置づけ | 内容 |
|---|---|---|---|
| 01 | [dhclient への切り替え](docs/01-dhclient.md) | 前提 | dhcpcd を止めて dhclient を systemd で動かす。ひかり電話の情報を DHCP オプションで受け取るための準備 |
| 02 | [Route53 による DDNS](docs/02-route53-ddns.md) | 関連 | Raspberry Pi での PPPoE 接続時に、aws CLI で Route53 の A レコードを自動更新する |
| 03 | [ひかり電話への直収](docs/03-hikari-denwa.md) | **中心** | DHCP で情報取得 → Asterisk をソースからビルド → pjsip.conf / extensions.conf を設定 |
| 04 | [携帯電話の Bluetooth 収容](docs/04-bluetooth-mobile.md) | 発展 | chan_mobile で iPhone を収容し、SIM の回線から発信する |

## 設定ファイル

ドキュメント中で使う設定ファイルを [`config/`](config/) に実ファイルとして置いています。

```
config/
├── asterisk/
│   ├── pjsip.conf          … ひかり電話直収の設定（03）
│   ├── extensions.conf     … ダイヤルプラン（03, 04）
│   └── chan_mobile.conf    … Bluetooth 携帯電話の設定（04）
├── dhcp/
│   └── dhclient.conf       … NGN 向け DHCP オプションの設定（03）
├── systemd/
│   ├── dhclient.service    … dhclient のユニット定義（01）
│   └── pulseaudio.service  … PulseAudio のシステムデーモン化（04）
├── pulse/
│   └── system.pa.append    … PulseAudio に追記する Bluetooth モジュール設定（04）
├── dbus/
│   └── pulseaudio-system.conf … PulseAudio から BlueZ へのアクセス許可（04）
├── ppp/
│   └── ip-up.local         … PPPoE 接続時に Route53 を更新するスクリプト（02）
└── aws/
    └── iam-policy.json     … Route53 更新専用の IAM ポリシー（02）
```

### 置き換えが必要な値

公開用に、環境固有の値は以下のようにマスクしています。自分の環境の値に置き換えてください。

| 表記 | 意味 | 取得元 |
|---|---|---|
| `0xxxxxxxxx` | ひかり電話の電話番号 | DHCP の `ntt.number` |
| `124.xxx.xxx.1` | ひかり電話の SIP サーバ | DHCP の `ip-sip-servers` |
| `ntt-west.ne.jp` | SIP ドメイン（西日本の例） | DHCP の `ntt.domain` |
| `xx:xx:xx:xx:xx:xx` | Bluetooth アダプタ／携帯電話のアドレス | `hcitool dev` / `mobile search` |
| `xxx.example.com` | DDNS で更新するホスト名 | 自分のドメイン |
| `ZXXXXXXXXXXXXX` | Route53 のホストゾーン ID | AWS コンソール |
| `ddns-updater` | aws CLI のプロファイル名 | `aws configure --profile` |

## 動作確認環境

執筆当時の環境です。現行バージョンでの動作は確認していません。

| 項目 | バージョン等 |
|---|---|
| ハードウェア | Raspberry Pi 2 Model B |
| OS | Raspbian 10 (buster), 32bit |
| Asterisk | 18.2.0（ソースからビルド） |
| 回線 | フレッツ光ネクスト（NTT 西日本）＋ひかり電話 |
| Bluetooth | Buffalo BSBT4D09BK（USB ドングル） |
| 携帯電話 | iPhone 5s（発信用 SIM を装着） |
| 確認時期 | 2021 年 2 月（01〜03）、2023 年 2 月（04） |

## 注意事項

- ここに掲載した構成や設定がベスト、あるいは唯一の正解であるとは考えていません。動作した一例としての参考情報です。
- 法律や NTT の約款との関係は考慮していません。NGN への直接接続は自己責任で行ってください。
- 参考リンクは執筆当時のものです。Asterisk の公式ドキュメントなど、移転しているものがあります。

## 元記事

| 本リポジトリ | Qiita 元記事 | 投稿日 |
|---|---|---|
| [01](docs/01-dhclient.md) | [Raspberry Pi(Debian)でdhclientを使う](https://qiita.com/kmorimoto/items/e76047be70dd64c08a1e) | 2021-02-18 |
| [02](docs/02-route53-ddns.md) | [Raspberry Pi(Debian)でRoute53を更新する](https://qiita.com/kmorimoto/items/77ea0370ee46f7b3e006) | 2021-02-17 |
| [03](docs/03-hikari-denwa.md) | [NTT光ネクストのひかり電話へAsteriskを直接接続する](https://qiita.com/kmorimoto/items/d99cd9edcf7436eea7cc) | 2021-02-19 |
| [04](docs/04-bluetooth-mobile.md) | [Asteriskで発信用の携帯電話をBluetoothで収容する](https://qiita.com/kmorimoto/items/3390c7c6565fcf249dfd) | 2023-02-23 |

## 謝辞

ひかり電話直収にあたっては、[voip-info.jp](http://www.voip-info.jp/) の情報、5ch.net および Asterisk 公式コミュニティでの議論を大いに参考にしました。感謝します。

## License

[MIT](LICENSE)
