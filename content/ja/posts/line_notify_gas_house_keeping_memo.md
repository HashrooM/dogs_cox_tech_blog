---
title: "LINE Notifyでお買い物備忘メモを送信する"
date: 2021-06-10T21:39:46+09:00
description: "LINE NotifyとGASを用いて買い物の備忘メモをLINEに送信します。"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: DogsCox
tags:
- GAS
- LINE Notify
series:
- 
categories:
-  "チャットボット"
---

この記事では [Google Apps Script](https://workspace.google.co.jp/intl/ja/products/apps-script/ "gas") (GAS) と [LINE Notify](https://notify-bot.line.me/ja/ "line_notify") を用いて
簡単な備忘メモを定期的にLINEに送信して通知するデモを紹介します。  

今回は自分が買い物をする時に忘れがちな品物をリマインダーとして通知するようにします。  
シャンプーの詰め替えとか忘れがちですよね...（私だけ？？）  
こういった品物を自分が買い物によく行く時間に合わせてリマインドするようにしてみます。  


## 用意するもの

- Googleアカウント
  - 特にスプレッドシートを使います
- LINEアカウント


## GASとは
[GAS](https://workspace.google.co.jp/intl/ja/products/apps-script/ "gas") 自体はGoogleが開発しているWeb上で動作するプログラムを作るためのプログラミング言語です。  

JavaScriptという言語がベースになってはいますが、Googleが提供する様々なサービス（Gメール、スプレッドシート、Googleマップ等々）と簡単に連携できるよう拡張が施されています。  

プログラミングに初めて触れる、という方でも簡単に便利ツールを作成できるようになるのでオススメです。  
GASは書いたプログラムを一行一行実行するスクリプト言語なので、この後はGASのプログラムのことをスクリプトと呼びます。  

GASは __無料__ で使えます。素晴らしいですね。  

ただし、実行には様々な制限があります。[Googleサービスの割り当て](https://developers.google.com/apps-script/guides/services/quotas "gas_quotas")  
詳細はリンク先を確認していただきたいのですが、多くの方に関係する制限としては以下の2つでしょうか。

- スクリプトの実行時間は __上限6分__ 。
- ファイル容量の制限あり。
  - 扱うサービスごとに上限は異なりますが、画像を送るには厳しいかな、と思えるくらいの制限です。（画質にもよりますが。）

とはいえ今回のような簡単なテキストを送信する程度であれば制限が気になることはないはずです。


## LINE Notifyとは
[LINE Notify](https://notify-bot.line.me/ja/ "line_notify") とは外部サービスからの通知をLINEのメッセージとして送信できるLINEの機能です。  

リンク先にもいくつか例が載っていますが、天気をお知らせしてくれるなどは分かりやすい例ですね。  

アクセストークンを取得するだけで実行できるので、お手軽に自身のLINEを便利ツールに返信させることが可能です。  
こちらも __無料__ です。
ちなみにアクセストークンとはパスワードのような認証のための単なる文字列です。


## 手順
今回はLINEにメッセージを送るだけの簡単なものを作ります。  
以下の手順です。  

1. LINE NotifyでLINEにメッセージを送信する際のアクセストークンを取得する。
1. スプレッドシートにリマインドしたい品物を記述する。
1. GASでスプレッドシートから品物を抽出し、LINEにメッセージとして送信するプログラムを書く。
1. GASのトリガーで定期的にリマインドを送信するように設定する。  

それでは、手順を一つ一つ説明していきます。  


## LINE Notifyのアクセストークンを取得する
まずは外部サービスから自身のLINEアカウントにメッセージを送るために、LINE Notifyのアクセストークンを取得します。  

[LINE Notify](https://notify-bot.line.me/ja/ "line_notify") にアクセスして、右上の "ログイン" から自身のLINEアカウントにログインします。  
(LINEってそうそうログインしないのでパスワード忘れがちです...)  

![LINE Notify account](/images/line_notify_gas_house_keeping_memo/line_notify_account.png "line_notify_account")  

ログインするとアクセストークンを発行することができます。図の "トークンを発行" をクリックします。  

![LINE Notify Token](/images/line_notify_gas_house_keeping_memo/line_notify_token.png "line_notify_token")  

トークン発行画面では以下のように発行するトークンの名前（アプリの名前と思って大丈夫です）とメッセージを送信するトークルームを選択できます。  

![LINE Notify Token publish](/images/line_notify_gas_house_keeping_memo/publish_token.png "lne_ntify_token_publish")  

今回はトークンの名前は "okaimonomemo" としました。  
また、今回のメッセージは自分用に送るだけなので、 "1:1でLINE Notifyから通知を受け取る" を選択します。  

"発行する" をクリックすると、トークンが発行されます。  
トークンは一旦どこかにメモしておきましょう（あくまで一時的に）。  
トークンが書かれた画面は一度消すと二度と同じトークンは発行されません。  

トークンをゲットしたらLINEでの設定は終了です。  
取得したアクセストークンは後ほどGASのスクリプト内に記述します。  


## スプレッドシートにリマインドしたい品物を記述
スプレッドシートにリマインドしたいものを書いていきます。  
これはGoogleドライブを開いてスプレッドシートを作るだけです。  

今回はちょうど部屋で切れていたシャンプーの詰替えを書いておきました。他は適当です。  
ただ品物を書くだけでは味気ないので、ついでにどこで買うかという情報も追加しました。  

![spreadsheet](/images/line_notify_gas_house_keeping_memo/spreadsheet.png "spreadsheet")  

さて、スプレッドシートでの作業はここまでなのですが、次にGASのスクリプトを書いていくので、その画面に移動しましょう。  
画像の赤矢印で示した "拡張機能" から "Apps Script" をクリックしてGASの画面を開いておきます。  


## GASでスクリプト記述
GASの画面を開くと最初は以下のようになっていると思います。  
（もしかしたら開くときに権限の認証を要求されるかもしれません。その時は認証します。）  

![gas init](/images/line_notify_gas_house_keeping_memo/gas_init.png "gas_init")  

"無題のプロジェクト" と書いてあるところにプロジェクトの名前を設定できます。  
ここでは "line_notify_house_keepings" としておきました。  

また、画面で "コード.gs" となってるところを右クリックするとスクリプトの名前を変えることができます。

さて、ではスクリプトを書いていきましょう。  


### スプレッドシートから情報を取得する関数
まずは先ほど作成したスプレッドシートから "買うもの" と "買う場所" を取得します。  

``` GAS
// スプレッドシートから買い物リストを抽出する関数
function getArrayFromSpreadsheet() {
  // 現在選択しているシートを取得
  const sheet = SpreadsheetApp.getActiveSheet();
  // 選択したシートから "買うもの" 全てを1次元配列として取得
  const what = sheet.getRange(2, 1, sheet.getLastRow() - 1).getValues().flat();
  // 選択したシートから "買う場所" 全てを1次元配列として取得
  const where = sheet.getRange(2, 2, sheet.getLastRow() - 1).getValues().flat();

  // 買うものと買う場所を配列にまとめて返す
  return [what, where]
}
```  

`SpreadsheetApp.getActiveSheet()` で現在選択中のシートのオブジェクトを取得します。  
次に `getRange()` で情報を取得する範囲を決めます。`getRange()` の詳細は [こちら](https://developers.google.com/apps-script/reference/spreadsheet/sheet?hl=jp#getRange(Integer,Integer,Integer) "getRange")。  

ここで、 `getLastRow()` で書き込まれている最終行を取得することで、書き込みがある最後の行まで全ての情報を取得します。  

最後の `flat()` は `getRange()` でシートの内容を取得すると何故か2次元配列になったので、1次元配列に直すために書いています。  

最後に買うものを買う場所をリスト形式で返してこの関数は完成です。  


### 取得した情報を送信用の文字列に整形する関数
スプレッドシートの情報を取得できたので、その情報をLINEのメッセージとして送信する文字列に整形します。  
これはお好みの文字列で良いのですが、今回は以下のようにしました。  

``` GAS
// 配列に格納されている買い物リストをLINEに送信するメッセージに加工する関数
function arrayToText(what_where) {
  // 買うものリストと買う場所リスト
  const what = what_where[0];
  const where = what_where[1];

  // 配列からメッセージを生成
  let message = "\n買うもの (買う場所)\n====================";
  for (let i=0; i<what.length; i++) {
    message += "\n" + what[i] + " (" + where[i] + ")"
  }

  return message
}
```  

"買うもの（買う場所）" の形式で一行一行記載しています。  


### LINEにメッセージを送信する関数
ここはLINE Notifyのドキュメントを参考に設定します。[LINE Notify Document](https://notify-bot.line.me/doc/ja/ "line_notify_document")  

おそらく "LINE Notify 使い方" などでググれば色々と先人の知恵をお借りすることができると思います。  
ただ、こういったAPIはいつの間にか仕様が変わっているものなので、何かしらエラーになったら公式のドキュメントを参照したほうが良いかもしれません。  

以下のスクリプト中の `LINE_NOTIFY_TOKEN` の `xxxxx` のところに先程LINE Notifyから取得したアクセストークンを記載します。  

``` GAS
// LINE notifyへメッセージを送信する関数
function sendToLINENotify() {
  const url = "https://notify-api.line.me/api/notify";
  const LINE_NOTIFY_TOKEN = "xxxxx";
  const message = arrayToText(getArrayFromSpreadsheet());
  const options =
  {
    "method": "post",
    "payload": {"message": message},
    "headers": {"Authorization": "Bearer " + LINE_NOTIFY_TOKEN}
  }

  // LINE notifyへメッセージとして送信
  UrlFetchApp.fetch(url, options);
}
```  


### スクリプト全体
ここで書いたスクリプトの全体です。  
最後の `sendToLINENotify()` 関数が本体となります。  

``` GAS
// スプレッドシートから買い物リストを抽出する関数
function getArrayFromSpreadsheet() {
  // 現在選択しているシートを取得
  const sheet = SpreadsheetApp.getActiveSheet();
  // 選択したシートから "買うもの" 全てを1次元配列として取得
  const what = sheet.getRange(2, 1, sheet.getLastRow() - 1).getValues().flat();
  // 選択したシートから "買う場所" 全てを1次元配列として取得
  const where = sheet.getRange(2, 2, sheet.getLastRow() - 1).getValues().flat();

  // 買うものと買う場所を配列にまとめて返す
  return [what, where]
}

// 配列に格納されている買い物リストをLINEに送信するメッセージに加工する関数
function arrayToText(what_where) {
  // 買うものリストと買う場所リスト
  const what = what_where[0];
  const where = what_where[1];

  // 配列からメッセージを生成
  let message = "\n買うもの (買う場所)\n====================";
  for (let i=0; i<what.length; i++) {
    message += "\n" + what[i] + " (" + where[i] + ")"
  }

  return message
}

// LINE notifyへメッセージを送信する関数
function sendToLINENotify() {
  const url = "https://notify-api.line.me/api/notify";
  const LINE_NOTIFY_TOKEN = "xxxxx";
  const message = arrayToText(getArrayFromSpreadsheet());
  const options =
  {
    "method": "post",
    "payload": {"message": message},
    "headers": {"Authorization": "Bearer " + LINE_NOTIFY_TOKEN}
  }

  // LINE notifyへメッセージとして送信
  UrlFetchApp.fetch(url, options);
}
```


## テスト実行
ここまでできたら一度テストとしてスクリプトを実行してみましょう。  
エディタ画面の上部メニューバーから関数 `sendToLINENotify` を選択し、 "実行" をクリックします。  

実行するためにまた何度か認証を行う必要があるかもしれません。都度認証します。  

うまく行くと自身のスマホのLINEに以下のようなメッセージが届きます。  

![LINE screenshot](/images/line_notify_gas_house_keeping_memo/line_screenshot.png "line_screenshot")  

(まあ大抵一発ではうまく行かないんですけどね)  


## トリガーを設定して定期実行する
最後に書いたスクリプトを定期的に実行するためのトリガーを設定します。  
自分が買い物に行くのが大体仕事終わりの19時以降なので、19時前にリマインドのメッセージが届くように設定します。  

先程のGASの画面左側の "トリガー" を選択します。  

![gas trigger](/images/line_notify_gas_house_keeping_memo/gas_trigger.png "trigger")  

"トリガー" 画面において、右下の "+トリガーを追加" のボタンをクリックします。  
すると、以下のような画面が現れます。  

![trigger setup](/images/line_notify_gas_house_keeping_memo/gas_trigger_setup.png "trigger_setup")  

指定した時間となったら実行されるように、各々以下のように設定します。  

- 実行する関数を選択 : `sendToLINENotify`
- 実行するデプロイを選択 : "Head" (これは選択の余地がないと思います)
- イベントのソースを選択 : "時間主導型"
- 時間ベースのトリガーのタイプを選択 : "日付ベースのタイマー"
- 時刻を選択 : "午後6時 ~ 7時"
- エラー通知設定 : "毎日通知を受け取る" (ここはお好みで)

上記の内容で "保存" ボタンをクリックしてトリガーを追加します。  

今回は関数を1つ実行するだけなのでトリガーも1つですが、必要に応じて複数のトリガーを作成することもできます。  
初めに言ったようにGASは実行の制限がキツイので、場合によっては複数のトリガーに処理を分散するなどといった工夫が必要になるかもしれません。  

これにて全ての設定は終了です。  
設定した時間にLINEにメッセージが届くようになります。

## まとめ
今回は GAS + LINE Notify で簡単なリマインドをするツールを作成しました。  
似たようなことは ToDoリストを使えばできるような気もしますが... まあそこはデモということで。 

今回は買い物リストはスプレッドシートで管理しているので、品物の追加、削除はスプレッドシートにて手動で行う必要があります。  
ここはちょっと面倒ですね。  

どうせなら品物の追加や削除もLINE上でできたらちょっとはマシになりそうです。  

ここまでやるとなると通知を出す機能しか持たないLINE Notifyでは厳しそうですね。  
LINEには自作のWebアプリと連携するためのより強化された機能もあるので、そのうちそちらも試してみたいと思います。  
