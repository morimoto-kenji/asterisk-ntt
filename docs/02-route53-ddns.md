# 02. Route53 による DDNS

Raspberry Pi で直接 PPPoE 接続している場合に、接続のたびに変わるグローバル IP アドレスを Amazon Route53 の A レコードへ自動で反映する。外部からホスト名で自宅の Raspberry Pi にアクセスできるようにするための設定である。

> 元記事：[Raspberry Pi(Debian)でRoute53を更新する](https://qiita.com/kmorimoto/items/77ea0370ee46f7b3e006)（Qiita, 2021-02-17）

## 方針

- DNS は別件で Route53 を使っており、aws CLI で簡単に更新できるので、Route53 を DDNS として使う。
- PPPoE 接続の確立時に pppd から実行される `/etc/ppp/ip-up.local` で aws CLI を実行する。

個々の要素に新しい技術はないが、組み合わせて一式にまとめた情報が見当たらなかったので整理した。

## 手順

### 1. aws CLI を入れる

[AWS 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/install-cliv2-linux.html)で案内されている aws CLI v2 は、OS の Python に依存しないバイナリで ARM 版もある。ただし 64bit OS 前提のバイナリで、32bit の Raspbian では動かなかった（[aws/aws-cli#4943](https://github.com/aws/aws-cli/issues/4943)）。

そこで Python に依存する v1 を使う。OS 全体の Python 環境を汚さないよう、pip で直接入れるのではなく、virtualenv にまとめて入れるバンドル版を使う。

```sh
curl -s "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
unzip awscli-bundle.zip
cd ./awscli-bundle
./install -i /usr/local/aws -b /usr/local/bin/aws
```

### 2. 更新専用の IAM ユーザを作る（任意）

既存のユーザや FullAccess 権限のユーザでも動くが、安全のため、対象のホストゾーンのレコード更新だけを許可したポリシーを作り、そのポリシーだけを持つ IAM ユーザを作る。

[`config/aws/iam-policy.json`](../config/aws/iam-policy.json)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "route53:ChangeResourceRecordSets",
            "Resource": [
                "arn:aws:route53:::hostedzone/ZXXXXXXXXXXXXX"
            ]
        }
    ]
}
```

ホストゾーン ID は AWS コンソールの Route53 で確認できる。作ったユーザのアクセスキーを、aws CLI のプロファイルとして登録する。

```sh
aws configure --profile ddns-updater
```

### 3. PPPoE 接続時に実行されるスクリプトを作る

`/etc/ppp/ip-up` の冒頭には、次のように書かれている。

```sh
# This script is run by the pppd after the link is established.
# It uses run-parts to run scripts in /etc/ppp/ip-up.d, so to add routes,
# set IP address, run the mailq etc. you should create script(s) there.
```

また途中に次の処理がある。

```sh
# This script can be used to override the .d files supplied by other packages.
if [ -x /etc/ppp/ip-up.local ]; then
    exec /etc/ppp/ip-up.local "$@"
fi
```

つまり `/etc/ppp/ip-up.local` を置いて実行権限を付ければ、PPPoE 接続の確立時に実行される。

[`config/ppp/ip-up.local`](../config/ppp/ip-up.local) → `/etc/ppp/ip-up.local`

```bash
#! /bin/bash

DIR="/etc/ppp"
JSON="$DIR/ip-up.json"
NEW_IP=$PPP_LOCAL

update_route53 () {
cat <<EOF > $JSON
{
  "Comment": "Updated by /etc/ppp/ip-up.local",
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "$1",
        "Type": "A",
        "TTL": 3600,
        "ResourceRecords": [
          {
            "Value": "$NEW_IP"
          }
        ]
      }
    }
  ]
}
EOF
/usr/local/bin/aws route53 change-resource-record-sets --hosted-zone-id $2 --change-batch file://$JSON --profile ddns-updater
}

NAME="xxx.example.com"
HOSTZONE_ID="ZXXXXXXXXXXXXX"
update_route53 $NAME $HOSTZONE_ID
```

```sh
chmod +x /etc/ppp/ip-up.local
```

pppd から渡される `PPP_LOCAL`（取得したグローバル IP アドレス）を使って変更内容の JSON を作り、aws CLI で Route53 の A レコードを `UPSERT` している。

JSON ファイルの置き場所が決め打ちになっているが、実用上の問題はないのでこのままにしている。
