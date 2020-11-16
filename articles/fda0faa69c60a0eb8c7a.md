---
title: "WPプラグイン「All-in-One WP Migration」を使って簡単にWordPressをデプロイする"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
前回の記事でAWSを用いてWordPressを立ち上げたので今回は既存WordPressサイトをそちらに移していきます。
**前回の記事はこちら**
[AWS初心者がWordPressをEC2&RDSで立ち上げる](https://zenn.dev/ichigo_dev/articles/2716e8508e25686fa8a6)
# 今回の実装内容
- さくらサーバで運用していたWordPressサイトを新しく構築したAWS上のWordPressに移行する。
- WPプラグイン「All-in-One WP Migration」を使用する。
- WPプラグイン「All-in-One WP Migration」の転送容量の制限を解除する。
- トップページ以外404エラーになる問題を解決する。
# 事前準備
移行したいサイトと移行先のサイトをそれぞれ用意します。
この際、移行先のサイトではWordPressをインストールするだけで他は何も触らなくてOKです。
（移行したら上書きされるため。）
# WPプラグイン「All-in-One WP Migration」のインストール
移行するサイト、移行先のサイトの両方にWPライブラリ、**All-in-One WP Migration**をインストールします。
![](https://storage.googleapis.com/zenn-user-upload/6etkxfetqylsbdh21fra0mde5wq8)
インストール出来たら有効化します。
左横のメニューバーにAll-in-One WP Migrationと表示されれば成功です。
# WordPressデータをエクスポートする
**移行するサイト側の作業になります。**
左横のメニューバーにある「All-in-One WP Migration」から**エクスポート**と進みます。
エクスポート先からファイルを選びダウンロードします。
![](https://storage.googleapis.com/zenn-user-upload/on32gya6cm11qmkwfx71lkkl7xis)
ファイルがダウンロード出来たら成功です。
# WPプラグイン「All-in-One WP Migration」の転送容量の制限を解除する
移行先サイトで先ほどダウンロードしたファイルからデータをインポートするのですが、デフォルトだと2MBまでしか転送できません。これを10GBまで転送が出来るようにします。
**移行先サイト側の作業になります**
### 専用プラグインをインストールする
1. 下記のリンクから専用のプラグインをダウンロードします。
https://import.wp-migration.com
![](https://storage.googleapis.com/zenn-user-upload/n3fy32idgi258vk186h2ec67xbh2)
Downloadを押してあげればOKです。
2. ダウンロードしたプラグインを移行先サイトにインストールする。
プラグイン＞新規追加ページのページ上部にある**プラグインのアップロード**から1でダウンロードしたフォルダをアップロードします。（展開しなくて大丈夫です。）
![](https://storage.googleapis.com/zenn-user-upload/uysy9po26cfj8vvzgjwmojvyltyp)
1. 確認
All-in-One WP Migration＞インポートで容量を確認してあげると容量が10GBまで増えています。
![](https://storage.googleapis.com/zenn-user-upload/7uurucz4l92upz077dvlhrzz0q42)
# WordPressデータをインポートする
**移行先サイト側の作業になります**
All-in-One WP Migration＞インポートを開きます。
インポート元で先ほどダウンロードしたファイルを選択し、インポートします。
![](https://storage.googleapis.com/zenn-user-upload/44ipy4pe17a4wp17fqbt9r8m30ix)
以上で完了です。
![](https://storage.googleapis.com/zenn-user-upload/8cuc6re9xnt2tcox4ez3jwnd9xop)
# メモ
- インポートは出来たけど、パーマリンク設定を基本にしないと404エラーが出ます。解決できていないので誰か教えてくれるとありがたいです。
![](https://storage.googleapis.com/zenn-user-upload/wkol0a2vsyei7hjqasui4tpdnjos)