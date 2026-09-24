# 01. dhclient への切り替え

Raspberry Pi（Raspbian / Raspberry Pi OS）で標準の DHCP クライアント [dhcpcd](https://roy.marples.name/projects/dhcpcd/) を止め、[dhclient（ISC DHCP）](https://www.isc.org/dhcp/) を systemd で動かす。

ひかり電話の情報（SIP サーバのアドレスや電話番号）は、フレッツ網（NGN）から DHCP オプション（[RFC 3361](https://tools.ietf.org/html/rfc3361) など）で配布される。この DHCP オプションを受け取るための準備として、DHCP クライアントを入れ替える。

> 元記事：[Raspberry Pi(Debian)でdhclientを使う](https://qiita.com/kmorimoto/items/e76047be70dd64c08a1e)（Qiita, 2021-02-18）

## 背景：なぜ dhclient なのか

通常の用途なら dhcpcd で問題はない。dhclient に替えた理由は、**dhcpcd で任意の DHCP オプションを扱う方法が見つからなかった** ためである。

1. **dhcpcd**：公式サイトのマニュアルが当時は公開停止中（"MAN PAGE IS CURRENTLY OFFLINE"）で、コンソールの `man dhcpcd` を一通り読んでも、該当する記述を見つけられなかった。
2. **NetworkManager**：最近の主流として検討した。[公式ドキュメント](https://developer.gnome.org/NetworkManager/stable/NetworkManager.conf.html)には、内蔵 DHCP クライアント（`dhcp=internal`）について "which is not currently as featureful as the external clients" と書かれている。実際に apt で入れて試すと、`/etc/dhcp/dhclient.conf` で詳細を設定すれば各種 DHCP オプションを扱えた（設定は `/etc/NetworkManager/conf.d/` に反映される）。
3. **dhclient**：ただ、NetworkManager を入れても結局 `dhclient.conf` で設定するのであれば、最初から入っている dhclient を直接動かせば済む。

以上から dhclient を採用した。

## 手順

### 1. dhcpcd を止める

```sh
systemctl stop dhcpcd.service
systemctl disable dhcpcd.service
```

[公式ドキュメント](https://www.raspberrypi.org/documentation/configuration/tcpip/)に従って `/etc/dhcpcd.conf` で固定 IP を設定している場合は、その設定を `/etc/network/interfaces.d/` へ移す必要がある。

### 2. dhclient のユニット定義ファイルを作る

DHCP クライアントはリース更新のために常駐させる必要があるので、systemd のユニット定義ファイルを作る。内容は [corvax19/dhclient.service（Gist）](https://gist.github.com/corvax19/6230283) をそのまま使っている。

[`config/systemd/dhclient.service`](../config/systemd/dhclient.service) → `/etc/systemd/system/dhclient.service`

```ini
[Unit]
Description=DHCP Client
Documentation=man:dhclient(8)
Wants=network.target
Before=network.target

[Service]
Type=forking
PIDFile=/var/run/dhclient.pid
ExecStart=/sbin/dhclient

[Install]
WantedBy=multi-user.target
```

インターフェースの起動順などでタイミングの調整が必要なら、`Before` / `After` で調整する。本構成では不要だった。

### 3. 有効化して起動する

```sh
systemctl daemon-reload
systemctl enable dhclient.service
systemctl start dhclient.service
```

### 4. 取得結果を確認する

DHCP で情報を取得できると、`/var/lib/dhcp/dhclient.leases` に内容が書き出される。

```
lease {
  interface "eth0";
  fixed-address xxx.xxx.xxx.xxx;
  option subnet-mask 255.255.255.xxx;
  option dhcp-lease-time 14400;
  option routers xxx.xxx.xxx.xxx;
  option dhcp-message-type 5;
  option dhcp-server-identifier xxx.xxx.xxx.xxx;
  option dhcp-renewal-time 7200;
  option dhcp-rebinding-time 10800;
  (略)
  renew 3 2021/02/17 05:11:02;
  rebind 3 2021/02/17 06:14:29;
  expire 3 2021/02/17 07:14:29;
}
```

ひかり電話向けの DHCP オプションを受け取る設定（`dhclient.conf`）は、[03. ひかり電話への直収](03-hikari-denwa.md#1-フレッツ網から-dhcp-で情報を取得する)で行う。

## 補足：rfkill のメッセージ

環境によるかもしれないが、Raspberry Pi 2B で上記の作業をすると、ログイン時に次のメッセージが出るようになった。

```
rfkill: cannot open /dev/rfkill: 許可がありません
rfkill: cannot read /dev/rfkill: 不正なファイル記述子です
```

[公式フォーラム](https://www.raspberrypi.org/forums/viewtopic.php?t=274816)でも同様の報告がある。2B（無線 LAN を内蔵していない）では実害がないため、メッセージを出しているスクリプトを無効化して対処した。

```sh
mv /etc/profile.d/wifi-check.sh /etc/profile.d/wifi-check.sh.bak
```
