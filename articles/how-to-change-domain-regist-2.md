---
title: "ドメインをさくらインターネットからRoute 53へダウンタイムなしで移管する方法-Part2"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ドメイン,移管,さくらインターネット,Route53]
published: true
---


## はじめに

前回の記事は[こちら][last-article]をご参照ください。

**今回、AWSのACM,CloudFront,S3を使用します。基本的に基本的にこの通りに設定すれば一定アクセスまでは無料のはずですが、自分がAWS初学者なのもあって料金体系の把握ができていなく、リクエストに対してコストがかかってしまう場合もございます。各自が提供するサービスのリクエストに応じて変化するのでかならず事前に調べるようにしてください。**
[AWS 料金表ページ][AWS-cost]

### 自分の目標

最終的にはこの構成図の通りになるようにしたいです
![domain-diagram](/images/howto-change-domain-regist/AWS-configDiagram-Web.png)

### 前回のあらすじ

ドメイン移管の準備としてとしてゾーン情報をDNSサーバーをAWSに移行しました。

#### 現状

- ドメインの管理はまださくらのドメインのまま(課金先はさくらのドメインのまま)。
- ドメインのゾーン情報,DNSのみAWSに移行した。
- APEXドメイン(kamifubuki-ao.com)にアクセスするとさくらのレンタルサーバでホスティングされているwebサイトは表示されるもののSSL証明書はLet's Encryptのまま

**だと思いますが、思考錯誤の過程でトライ&エラー繰り返してしまったので同じように作業しているあなたが全く同じ状況かわかりません。最終的に設定が終われば問題なく動作しますが、途中で記事に書いてるにもかかわらずドメインに繋がらなくなった等ございましたら以下問い合わせにてご連絡いただくかTwitterのDMまでご連絡ください。**
[お問い合わせ][contact-link]

## ドメインを移管する

[Part1][last-article]で準備が終わったので、今回の一番の目的であるドメインの移管を行っていきます。

### さくらのドメインで移管手続きをする

ドメインの移管には、AuthCode(オースコード)という認証コードが必要になりますので、それをさくらのドメインに発行してもらう必要があります。
会員メニュー > 契約中のドメイン一覧 > 移管したいドメインの手続きの欄に"転出"のリンクがあるのでそこから問い合わせます。

![sakura-domain-1](/images/how-to-change-domain-regist-2/sakura-1.png)

同意してください

![sakura-domain-2](/images/how-to-change-domain-regist-2/sakura-2.png)

一番上、会員基本情報のメールアドレスに発行されたAuthCodeが届くので、必ず受け取れるメールアドレスにしてください。

サービス名称はすでにドメイン取得が選択されているので、変更しなくて大丈夫です。

サーバー名/ドメイン名/IPアドレス/サービスコードの欄は任意になるので記入してもしなくても大丈夫ですが、一応記入しました。

その下、質問内容のgTLDドメイン移管(転出依頼)はすでにすでに選択されていると思います。
お問い合わせ区分は新たに実施~で大丈夫です

![sakura-domain-3](/images/how-to-change-domain-regist-2/2025-03-11-15-24-25.png)

申請ドメイン名も記述済みなので大丈夫です。
管理者メールアドレスは、Route 53で申請したときにVerifyメールが来るので、受け取れるアドレスが良いですが、**もしゾーン情報のコピーミスでメールが自分のアドレスに届かなくなる可能性があるので、もし不安な方は他のメールサービスでも良いと思います。自分はiCloud.comで記入しています。**

下の方、解約理由等は任意になるのでどちらでも大丈夫です。

一番下の確認事項の同意だけチェックを忘れないようにして"次へ"をクリック。

内容を確認して"お問い合わせの送信"で送信してください。

回答までに2~3営業日かかるので一旦放置します。

## SSL/TLS証明書周辺の設定を行う

転出の手続きを待つ間に、構成図にも記載したようにSSL/TLS証明書周りの設定を行います。
今回目指す構成をもう一度おさらいしましょう。

![domain-diagram](/images/howto-change-domain-regist/AWS-configDiagram-Web.png)

いままでSSL/TLS証明書はさくらのレンタルサーバーで取得できたLet's Encryptで証明を行っていたのですが、今回の移管に合わせてすべてAWS内で完結させたかったのでこのような形を撮ることにしました。

Route 53のみではSSL/TLS証明書の管理ができないため、www.kamifubuki-ao.comとさくらのレンタルサーバの間にCloudFrontを噛ませてそことACMでSSL/TLS証明書を更新させるようにします。

### AWS Certificate Manager(ACM)でSSL/TLS証明書を作成する

この部分の設定を行っていきます。
![domain-diagram-focus-1](/images/how-to-change-domain-regist-2/AWS-configDiagram-Web-1.png)

[ACM][ACM]のコンソールから証明書一覧のページに飛びます

::: message alert
のちに後に作成するCloudFrontのリージョンがグローバルになるので、ACMのリージョンがus-east-1(米国(バージニア北部))なのを確認してください。
それ以外のリージョンで作成されたSSL/TLS証明書はCloudFrontで使用できません。
:::

リクエストから新しい証明書を作成します。
証明書タイプはパブリック証明書にしてください。

![new-SSL/TLS-certificate](/images/how-to-change-domain-regist-2/2025-03-11-17-17-52.png)

::: message alert
ここの完全修飾ドメイン名にkamifubuki-ao.comのみ入力すると、APEXドメインしか有効にならなくなってしまいます。
自分の場合個人的なこだわりで<www.つき>にリダイレクトさせたいので"*.kamifubuki-ao.com"のワイルドカードをつけたドメインも追加しておきます。

ほかのサイトでサブドメインを使用したい場合はワイルドカードを追加したドメインも追加するようにしてください。

もし<www.ドメイン>のみの利用想定であればワイルドカードではなく<www.ドメイン>の形で追加しても構いません。(他のサブドメインでの利用も考えているので自分はワイルドカードワイルドカードを使用しています。)
:::

そのほかのキーアルゴリズム,タグは初期設定のままで構いません。

### ゾーン情報にCNAMEレコードを追加する

AWSの場合、詳細画面からRoute 53で直接レコードを作成できるのでそちらから追加してしまって大丈夫です。便利。

![ACM](/images/how-to-change-domain-regist-2/2025-03-11-17:26:13.png)
![ACM-2](/images/how-to-change-domain-regist-2/2025-03-11-17:28.28.png)

上の画像は両方とも両方とも検証ステータスが成功になっていますが、追加したばかりだとなっていないはずなので、両方ともチェックを入れてレコードを作成してください。

### CloudFrontのディストリビューションを作成する

続いて、こちらの設定をしていきます。
![domain-diagram-focus-2](/images/how-to-change-domain-regist-2/AWS-configDiagram-Web-2.png)

[CloudFront][CF]のコンソールにアクセスします。
CloudFrontは自動的にリージョンがグローバルになるので、特にリージョンは設定しなくて大丈夫です。

一覧の右上にある"ディストリビューションを作成"から新しいディストリビューションを作成します。

![new-Dist](/images/how-to-change-domain-regist-2/2025-03-11-18-28-44.png)

| 項目名 | 設定内容 |
| --- | --- |
| Origin dmain| さくらのレンタルサーバの初期ドメイン |
| プロトコル | HTTPSのみ |
| Minimum Origin SSL protocol | TLSv1.2 |
| Origin Path | 空欄 |
| 名前 | 自由(後から変更できないのでなんでもいいですが慎重に) |
|カスタムヘッダーを追加 | 空欄 |
| Enable Origin Shield | いいえ |
| 追加設定 | すべて初期値 |

![new-Dist-2](/images/how-to-change-domain-regist-2/2025-03-11-19-46-45.png)

| 項目名 | 設定内容 |
| --- | --- |
| パスパターン | デフォルト |
| オブジェクトを自動的に変換 | Yes |
| ビューワープロトコルポリシー | Redirect HTTP to HTTPS |
| 許可されたHTTPメソッド | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE |
| ビューワーのアクセスを制限する | No |

![new-Dist-3](/images/how-to-change-domain-regist-2/2025-03-11-19-50-41.png)

| 項目名 | 設定内容 |
| --- | --- |
| キャッシュキーとオリジンリクエスト | Cache policy and origin request policy (recommended) (初期値) |
| キャシュポリシー/オリジンリクエストポリシー | 初期値(Recommended for custom origins) |
| レスポンスヘッダーポリシー | 空欄 |
| 追加設定 | すべて初期値 |

---

**次の項、WAFに関してはあなたが管理するサービスの重要度に応じて設定してください。**
**ここでのスクリーンショットを用いた説明はしません。(自分がオフで設定しているため)**
**セキュリティ保護を有効にするともっともベーシックな設定で1000万件のりクエストに対して14USDのコストがかかります。WAFをオフにする場合、追加コストはかかりません。**

---

![new-Dist-4](/images/how-to-change-domain-regist-2/2025-03-11-19-58-26.png)

| 項目名 | 設定内容 |
| --- | --- |
| Anycast static IP list | 空欄 |
| 料金クラス | すべてのエッジロケーションを使用する (最高のパフォーマンス) |
| 代替ドメイン名 (CNAME) | <www.つきのドメイン名> |
| Custom SSL certificate | 先ほどACMで作成した証明書を選択 |
| レガシークライアントサポート | チェック外したまま |
| セキュリティポリシー | TLSv1.2_2021 (推奨) |

![new-Dist-5](/images/how-to-change-domain-regist-2/2025-03-11-22-29-27.png)

| 項目名 | 設定内容 |
| --- | --- |
| サポートされているHTTPバージョン | 特に制限ないですが、両方チェックをいれておけば問題ないです |
| デフォルトルートオブジェクト | 空欄 |
| IPv6 | 各自がホスティングするサービスでIPv6を使用していなければオフ |
| ログ配信 | おまかせします |

以上設定したらディストリビューションを作成。

![Maked-Dist](/images/how-to-change-domain-regist-2/2025-03-11-23-41-15.png)

代替ドメイン名が自身が目指したいドメインになっているか、カスタムSSL証明書の右端に緑のチェックマークがあるかどうかを確認してください。

## <www.ドメイン>からのルーティングを行う

<www.ドメイン>からのリクエストをCloudFrontに経由させてオリジン(さくらのレンタルサーバ)に向かわせるようにRoute53の設定をしていきます。

下図の部分です。
![domain-diagram-focus-4](/images/how-to-change-domain-regist-2/AWS-configDiagram-Web-3.png)

まずRoute53の[コンソール][r53-console]から該当ドメインのホストゾーンのページを開きます。

レコードを作成
![r53-hostzone](/images/how-to-change-domain-regist-2/2025-03-12-0-17-14.png)

| 項目 | 設定内容 |
| --- | --- |
| レコード目 | www |
| レコードタイプ | A |
| エイリアス(トグルスイッチ) | オン |
| トラフィックのルーティング名 | CloudFrontディストリビューションへのエイリアス |
| 2個目のドロップダウンリスト | 先ほど作成したCloudFrontディストリビューションのドメイン名 |
| ルーティングポリシー | シンプルルーティング |

入力が完了したらレコードを作成してください。

Chrome等で開発者ツールのネットワークタブを開いたまま開いたまま確認して
x-cache:    Miss from cloudfront
の表記があればCloudFrontを噛ませられている証拠です。

![Chrome](/images/how-to-change-domain-regist-2/2025-03-12-02-18-04.png)

![SSL/TLS](/images/how-to-change-domain-regist-2/2025-03-12-02-20-11.png)

httpsで接続できていること/発行元の一般名がAmazon RSAになっていることが確認できれば
この章でやりたかったことは完了です。

## APEXドメインへのリクエストを<www.ドメイン>にリダイレクトさせる

今回、個人的にAPEX(裸)ドメインからのアクセスも<www.ドメイン>に合流させたいので、APEXドメインへのリクエストを<www.ドメイン>にリダイレクトさせる設定を行います。

この部分ですね。

![domain-diagram-focus-4](/images/how-to-change-domain-regist-2/AWS-configDiagram-Web-4.png)

AWS Route53では裸ドメインから<www.ドメイン>に飛ばすCNAMEレコードを作成することが仕様上できません。そのため、回避策としてS3バケットにて静的Webサイトをホスティングしてそちらから301リダイレクトを行います。

### S3バケットを作成する

::: message
作成するリージョンの縛りはACMと違ってありません。お好みのリージョンで作成してください。
:::

![S3-Bucket-1](/images/how-to-change-domain-regist-2/2025-03-12-02-33-33.png)

| 項目 | 設定内容 |
| --- | --- |
| バケットタイプ | 汎用 |
| バケット名 | APEXドメイン(リダイレクト元となるドメイン)|
| オブジェクト所有者 | ACL無効(推奨) |

ここから下はすべて初期値のままで大丈夫です。
一番下"バケットを作成"から作成してください。

::: message
バケット名はリダイレクト元になるドメインにしてください。
'www.kamifubuki-ao.com' > 'kamifubuki-ao.com'  → 'www.kamifubuki-ao.com'
'kamifubuki-ao.com' > 'www.kamifubuki-ao.com' → 'kamifubuki-ao.com'
:::

### S3バケットの静的ウェブサイトホスティング機能を用いて301リダイレクトを設定する

作成したバケット名をクリックして詳細ページを確認してください。
オブジェクトは無いままで大丈夫です。

プロパティタブの一番下、静的ウェブサイトホスティングの欄の編集をクリックします。

![S3-Bucket-2](/images/how-to-change-domain-regist-2/2025-03-12-02-44-16.png)

| 項目 | 設定内容 |
| --- | --- |
| 静的ウェブサイトホスティング | 有効にする |
| ホスティングタイプ | オブジェクトのリクエストをリダイレクトする |
| ホスト名 | <www.ドメイン> |
| プロトコル | https |

以上確認したら変更を保存します。

### Route53にてAPEXドメインへのリクエストを作成したS3バケットに飛ばす

Route53の[コンソール][r53-console]に戻ってホストゾーンを確認します。

[以前][last-article-r53]コピーしたときにAPEXドメインのAレコードをさくらのレンタルサーバに
向けて設定していたはずです。そこを先程作成したS3バケットででホスティングしているWebサイトにするだけです。

![R53](/images/how-to-change-domain-regist-2/2025-03-12-03-01-00.png)

| 項目 | 設定内容 |
| --- | --- |
| レコード名 | 空欄 |
| レコードタイプ | A |
| エイリアス(トグルスイッチ) | オン |
| トラフィックのルーティング先1 | S3ウェブサイトエンドポイントへのエイリアス |
| トラフィックのルーティング先2 | 先ほど作成したバケットが存在するリージョン |
| トラフィックのルーティング先3 | バケットを選択(APEXドメイン) |
| ルーティングポリシー | シンプルルーティング |
| ターゲットのヘルスを評価 | いいえ |

以上確認したら保存してください。

## さくらのインターネット側の設定を確認する

さくらのレンタルサーバ側で初期ドメインやWebサービスをホスティングしてるサーバー側の設定を最後行います。

::: message alert
Twitterで嘆きましたが、設定完了したと思ってAPEXドメインにアクセスしたところ、無限にリダイレクトされる現象が起きました。のちのち確認したところ、**さくらのレンタルサーバ側でWAFを設定していたのが原因でした。**

以下設定する際は注意してください。
:::

### ドメイン設定を更新する

![sakura-config](/images/how-to-change-domain-regist-2/2025-03-12-04-47-13.png)

さくらレンタルサーバ側で初期ドメインの設定を行います。(CloudFrontのオリジンとしてリクエストを送るため)

画像のなかで重要なのは以下の2点です。

- SSLの利用 → SSLを利用する
- HTTPS転送設定 → HTTPSに転送する

### さくらのレンタルサーバ側でWAFをオフにする

![WAF-off](/images/how-to-change-domain-regist-2/2025-03-12-20-45-19.png)

セキュリティ > WAF設定ドメインから初期ドメインの利用するのチェックを外してください。

::: message alert
この設定がオンのままだったからか、APEXドメイン,<www.ドメイン>どちらにアクセスしても無限にリダイレクトが繰り返される問題が起き続けました、、
推奨されていませんがさくらのレンタルサーバ側ではWAFがオンにできないようなので、WAFを使用する場合はRoute53から設定するようにしてください。
**追加コストがかかる場合があります。**
:::

### ドメインの移管を完了する

さくらのドメインから移管手続きをした後2~3営業日でさくらのドメインからAuth Codeが記載されたメールが届きます。そのAuth CodeをRoute53に入力したら移管申請が完了になります。

Route53のサイドペインからドメイン > 登録済みドメインを確認します。

![R53-1](/images/how-to-change-domain-regist-2/2025-03-12-20-51-57.png)
右上の移管(イン)から単一のドメインを選択。

あとは案内に従ってドメイン,事業者情報,Auth Codeを入力すれば申請は完了です。
あとは2,3営業日後にさくらのドメインで移管申請をしたときに入力したメールサービスに認証用リンクが届くのでアクセスして認証すすればドメインの移管は完了です。

### おつかれさまでした

設定するうえで、なにかお困りのことございましたらお気軽に下記リンクからお問い合わせください。対応いたします。
[お問い合わせ][contact-link]

[last-article]: https://zenn.dev/leopardkk/articles/howto-change-domain-regist-1 "前回の記事"
[last-article-r53]: https://zenn.dev/leopardkk/articles/howto-change-domain-regist-1#route-53%E3%81%AB%E3%81%A6%E3%83%9B%E3%82%B9%E3%83%88%E3%82%BE%E3%83%BC%E3%83%B3%E3%82%92%E6%96%B0%E8%A6%8F%E4%BD%9C%E6%88%90 "前回の記事"
[aws-cost]: https://aws.amazon.com/jp/pricing/?nc2=h_ql_pr_ln&aws-products-pricing.sort-by=item.additionalFields.productNameLowercase&aws-products-pricing.sort-order=asc&awsf.Free%20Tier%20Type=*all&awsf.tech-category=*all "AWSコスト"
[contact-link]: https://aoskyblue.notion.site/1a8c3a7e5a7b814cabc5ff8d80de75fa?pvs=105 "問い合わせフォーム"
[ACM]: https://us-east-1.console.aws.amazon.com/acm/home?region=us-east-1#/certificates/list "ACM"
[CF]: https://us-east-1.console.aws.amazon.com/cloudfront/v4/home?region=us-east-1#/distributions "CF"
[r53-console]: https://us-east-1.console.aws.amazon.com/route53/v2/hostedzones?region=us-east-1# "R53-Console"
