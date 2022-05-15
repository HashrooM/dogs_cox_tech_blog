---
title: "【備忘録】Chromebookのセットアップ：日本語化とdockerインストール"
date: 2021-06-15T22:08:47+09:00
description: "ChromebookのLinux環境を日本語化してDockerをインストールします。"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: DogsCox
tags:
- Chromebook
series:
- 環境構築
categories:
- Tips
---

この記事では chromebook で使える Linux環境 "crostini" の日本語化と Docker のセットアップ方法を解説します。  
個人的な作業の備忘録です。  

皆さん、 chromebook 使ってますか？  
安価な PC ながらブラウジングから簡単なプログラミングまでこなせる案外使えるヤツです。  

最近はメイン機としても chromebook を使っているのですが、  
chromebook 搭載の Linux 環境 "crostini" は日本語に対応しておらず、辛いなーと思っていたので  
この度とうとう日本語化に臨むことにしました。  

ついでに Docker もインストールしてみます。  
けっこう chromebook の開発環境作るのって試行錯誤すること多いんですよね...  
そこで Docker上に開発環境を乗っけることにしました。  


## 環境
使用している chromebook の環境です。  
OSのバージョンは設定時の最新にアップデートしておきました。  

Chromebook の環境構築は結構マシン依存する...? 気がします。  

| もの | 詳細 |
| ---- | ---- |
| マシン | ASUS Chromebox4 |
| CPU | Intel Core-i3-10110U (メモリ容量 8GB) |
| ストレージ | SSD 128GB |
| OSバージョン | ChromeOS バージョン: 91.0.4472.102（Official Build） （64 ビット） |

はい、chromebook ではなく chromebox でした。  

<!-- START MoshimoAffiliateEasyLink -->
<script type="text/javascript">
(function(b,c,f,g,a,d,e){b.MoshimoAffiliateObject=a;
b[a]=b[a]||function(){arguments.currentScript=c.currentScript
||c.scripts[c.scripts.length-2];(b[a].q=b[a].q||[]).push(arguments)};
c.getElementById(a)||(d=c.createElement(f),d.src=g,
d.id=a,e=c.getElementsByTagName("body")[0],e.appendChild(d))})
(window,document,"script","//dn.msmstatic.com/site/cardlink/bundle.js?20220329","msmaflink");
msmaflink({"n":"ASUS｜エイスース デスクトップパソコン Chromebox 4 ブラック CHROMEBOX4-G5020UN [モニター無し \/intel Core i5 \/メモリ：8GB \/SSD：128GB \/2021年5月モデル]","b":"","t":"","d":"https:\/\/thumbnail.image.rakuten.co.jp","c_p":"\/@0_mall\/biccamera\/cabinet\/product\/6531","p":["\/00000009256947_a01.jpg","\/00000009256947_a02.jpg","\/00000009256947_a03.jpg"],"u":{"u":"https:\/\/item.rakuten.co.jp\/biccamera\/0192876974735\/","t":"rakuten","r_v":""},"v":"2.1","b_l":[{"id":1,"u_tx":"楽天市場で見る","u_bc":"#f76956","u_url":"https:\/\/item.rakuten.co.jp\/biccamera\/0192876974735\/","a_id":2682976,"p_id":54,"pl_id":27059,"pc_id":54,"s_n":"rakuten","u_so":1}],"eid":"BHvls","s":"s"});
</script>
<div id="msmaflink-BHvls">リンク</div>
<br>
<!-- MoshimoAffiliateEasyLink END -->


## corostini の日本語化
色々なサイト様を参考にしたのですが、うまく行かないことが多々あり...  
最終的にこちらのサイト様を参考にしたらできました。  

[Crostiniで日本語環境設定:健康で文化的な最低限度の生活のためのChromebook設定(2)](https://scrapbox.io/hada/Crostini%E3%81%A7%E6%97%A5%E6%9C%AC%E8%AA%9E%E7%92%B0%E5%A2%83%E8%A8%AD%E5%AE%9A:%E5%81%A5%E5%BA%B7%E3%81%A7%E6%96%87%E5%8C%96%E7%9A%84%E3%81%AA%E6%9C%80%E4%BD%8E%E9%99%90%E5%BA%A6%E3%81%AE%E7%94%9F%E6%B4%BB%E3%81%AE%E3%81%9F%E3%82%81%E3%81%AEChromebook%E8%A8%AD%E5%AE%9A(2) "crostini_japanise")  

以下、ほぼ上記サイト様に書いてあることと同様ですが...  
バックアップとして手順だけメモしておきます。  

### 手順
(1) crostini ターミナル環境の日本語化

```
# タイムゾーンの変更
$ sudo dpkg-reconfigure tzdata
# 矢印キーやマウスで "Asia" -> "Tokyo" を選択 -> "OK"

# ロケールや日本語フォントのインストール
$ sudo apt -y install task-japanese locales-all fonts-ipafont

# ロケールの変更
$ sudo localectl set-locale LANG=ja_JP.UTF-8 LANGUAGE="ja_JP:ja"
$ source /etc/default/locale
```

(2) 日本語入力エンジン MOZC のインストール

```
$ sudo apt install fcitx-mozc
```

(3) fcitxの自動起動設定
`/etc/systemd/user/cros-garcon.service.d/cros-garcon-override.conf` に以下を追記。  

```
Environment="GTK_IM_MODULE=fcitx"
Environment="QT_IM_MODULE=fcitx"
Environment="XMODIFIERS=@im=fcitx"
```

`~/.sommelierrc` に以下を追記。  

```
/usr/bin/fcitx-autostart
```

(4) MOZC を入力メソッドに追加
以下のコマンドで入力メソッドの設定を起動。  

```
$ fcitx-configtool
```

GUIで設定ツールが立ち上がるので、入力メソッドに "MOZC 日本語" を追加。  
ここまでの設定がうまく行っていたら何もしなくてもすでに追加されていると思います。  


### crostini の日本語化：補足
どうも一番最初の `dpkg-reconfigure tzdata` をしないと自分の環境ではうまく行かなかったようです。  
このコマンドは最近の Debian 系統の OS のタイムゾーンを変更する方法らしいです。  

[Ubuntu 16.04 (Xenial) 以降のタイムゾーン設定方法](https://please-sleep.cou929.nu/xenial-localtime.html "debian_locale")  


## Docker セットアップ
Docker のセットアップは [公式ページ](https://docs.docker.com/engine/install/debian/ "docker_debian")に従うと普通にできました。  
注意点としてはUbuntu 用のセットアップではなく Debian 用のセットアップ方法にするというくらいでしょうか。  

Docker をインストールしたら自分を docker グループに追加して sudo なしで docker を使えるようにしましょう。  

```
$ sudo usermod -aG docker `whoami`
```

ついでに docker-compose もインストールしておきました。  
こちらも[公式ページ](https://docs.docker.com/compose/install/ "docker_compose") に従うと普通にできました。  

[前回の記事]({{<ref "/posts/ubuntu_setup.md" >}}) では python 環境を pyenv で構築しましたが、今後は docker にしようかなぁ...と考え中です。  