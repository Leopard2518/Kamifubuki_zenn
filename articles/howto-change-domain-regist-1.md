---
title: "ドメインをさくらインターネットからRoute 53へダウンタイムなしで移管する方法-Part1"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ドメイン,移管,さくらインターネット,Route53]
published: true
---

最近AWSを学習していることもあり、現在自分が所持している[ドメイン][mydomain-link]を[さくらインターネット][sakura-addr]から[Route 53][route-53-addr]に移管したので、その方法を備忘録として残そうと思います。

[ゾーン情報のコピー](#route-53にてホストゾーンを新規作成)でWhoisのゾーン情報をRoute 53にコピーしていなかったため、一時[独自サイト][mydomain-link]にアクセスできなくなるほか、同じドメインでさくらのレンタルサーバで運用していたメールサービスも使用できなくなる事故を起こしてしまったため、ゾーン情報コピーは忘れないようにしてください。

::: message alert
**解決しています**
現在サイトにアクセスできない事態は解決したものの、メールアドレスには一部メールが届かない事態が引き続き発生しています。解決方法がわからず奮闘中です
↓

解決しました。
後ほど記載しますが、さくらのドメインで記載のあった自ドメインを指す「＠」をRoute 53にコピーする際、そのまま「@」と入力してしまっていたのが原因でした。さくらのドメイン側にある「＠」はRoute 53に移す際「{独自ドメイン}.」と入力するようにしてください
:::

---

## はじめに

### 自分の状況

- [さくらのドメイン][domain-sakura-ne-jp]にて契約した独自ドメイン
- 上記ドメインは同じく[さくらのレンタルサーバ][rs-sakura-ne-jp]にて契約したサーバーに紐づけ
- さくらのレンタルサーバではWebページ,メールサービスを利用
- AWSのRoute 53へのドメイン移管をしたい
- AWSのリージョンはアジアパシフィック(東京)を使用。Route 53はグローバル
- 自分の管理しているアドレスはkamifubuki-ao.com
- 最終的にはAPEXドメイン→www.つきドメインにリダイレクトしたい
- SSL証明書の管理をAWS側でできるようにしたい

### 今回最終的に目指す構成

![config diagram](/images/howto-change-domain-regist/AWS-configDiagram-Web.png)

## Route 53にてホストゾーンを新規作成

![AWS console](/images/howto-change-domain-regist/AWS_Route53_1.png)
右上のホストゾーンの作成から新しい新しいホストゾーンを作成します。

- ドメイン名には自分が管理したいドメインのAPEXドメイン(裸ドメイン)を入力
- 説明欄は空欄で◯
- タイプはパブリックのホストゾーンを選択
- タグは特に必要なし

![AWS Console](/images/howto-change-domain-regist/2025-03-08-20-07-30.png)

完了後自動的にAPEXドメインのNSレコード,SOAレコードが作成されますので、触らすに一旦は放置で大丈夫です。

![AWS Console](/images/howto-change-domain-regist/1.png)

続いて、さくらのドメインで設定されているゾーン情報をRoute 53にコピーしていきます。

::: message
必ずコピーしないといけないゾーン情報(TTLはお好みで大丈夫です(自分は300s(5m)のままにしています))

| レコード名 | タイプ | 値 |
| ---- | ---- | ---- |
|kamifubuki-ao.com | CNAME | {IPv4 for sakura(Webserver)} |
| {domain} | MX | 10 {さくらのレンタルサーバの初期ドメイン}.sakura.ne.jp. |
| mail.{domain} | CNAME | {domain}. |
| ftp.{domain} | CNAME | {domain}. |

:::

::: message alert
冒頭にも記載しましたが、さくらのドメインでは独自ドメインをアライアスとして"@"表記しているようです。Route 53にそのままコピーせずに{domain}になおして入力してください。
EX.

### さくらのドメイン

| レコード名 | タイプ | 値 |
| ---- | ---- | ---- |
| ftp.kamifubuki-ao.com | CNAME | @ |

### Route 53

| レコード名 | タイプ | 値 |
| ---- | ---- | ---- |
| ftp.kamifubuki-ao.com | CNAME | kamifubuki-ao.com |

:::

## さくらのドメインのWhois情報に設定されているDNSをAWSのものに設定する

![Sakura domain](/images/howto-change-domain-regist/Sakura-domain.png)
中程にあるネームサーバを編集をクリックしてAWSでゾーンを作成して時に自動的に作成された4つのDNSに設定します。

![DNS-settings](/images/howto-change-domain-regist/dns-settings.png)

## ドメインの移管準備は完了

実際にリンクにアクセスして問題なく問題なく自分のサイトに接続できるか確認してください。
**キャッシュの影響で前のサイトが表示される可能性があるため、プライベートブラウザでの確認を推奨します**

## 次回

- APEXドメインからwww.つきドメインへのリダイレクト
- AWS側でSSL証明書を実装

次回: now on writing...

[mydomain-link]: http://www.kamifubuki-ao.com/ "独自管理ドメイン"
[sakura-addr]: https://www.sakura.ad.jp/ "さくらインターネットHP"
[route-53-addr]: https://aws.amazon.com/jp/route53/ "Route53HP"
[domain-sakura-ne-jp]: https://domain.sakura.ad.jp/ "さくらのドメインHP"
[rs-sakura-ne-jp]: https://rs.sakura.ad.jp/ "さくらのレンタルサーバHP"
