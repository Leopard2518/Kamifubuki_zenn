---
title: "開発環境にMisskeyのインスタンスを作成してみる"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Misskey","Ubuntu","Linux"]
published: false
---

## Misskeyとは

> Misskeyはフリーかつオープンなプロジェクトで、誰でも自由にMisskeyを使ったサーバーを作成できるため、すでに様々なサーバーがインターネット上に公開されています。
> また重要な特徴として、MisskeyはActivityPubと呼ばれる分散通信プロトコルを実装しているので、どのサーバーを選んでも他のサーバーのユーザーとやり取りすることができます。
> これが分散型と言われる所以で、単一の管理者によって単一のURLで公開されるような、Twitterなどのほかサービスとは根本的に異なっています。
> サーバーによって主な話題のテーマやユーザー層、言語などは異なり、自分にあったサーバーを探すのも楽しみの一つです。（もちろん自分のサーバーを作るのも一興です）。

@[card](https://misskey-hub.net)

Misskeyは、オープン・分散・高機能・高カスタマイズ性を売りにした分散型SNSで、twitterとDiscordの機能を足したようなSNSとなっています。

## 主な機能

### -ノート

Misskeyでは、ユーザーの投稿をノートと呼びます。他ノートの引用、画像動画等ファイルの添付もでき、イメージとしてはTweetです。

### -連合

分散通信プロトコルのActivityPubを使用しているため、他MisskeyサーバーをはじめActivityPubを使用している他のサービスとも連携することができます。

### -リアクション

Disscordにおけるリアクションと同じ機能です。絵文字を使って該当ノートに自分の気持ちを表現することができます。

### -ドライブ
ノートによってアップロードしたファイルを管理するインターフェイスがあります。
お気に入りの画像をまとめたり、再共有も可能です。

### -テーマ

自分の好きなデザインでMisskeyを使うことができます。
ダークモードはもちろん単色だけでなく自分でカスタマイズすることも可能です。

### -スレッド

Twitterと同じようにスレッドを作成してやり取りすることが可能です

### -チャート

組み込み式のチャートエンジンを備えているため、サーバーの利用状況が簡単に可視化できます。

### -ウィジェット

様々な種類のウィジェットを配置し、好みにUIをカスタマイズする必要ができます。


その他詳細は以下ドキュメントをご参照下さい。

@[card](https://misskey-hub.net/docs/misskey.html)