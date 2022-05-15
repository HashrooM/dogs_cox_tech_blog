---
title: "【Flask入門 1】Flaskで作る英単語学習アプリver0.1(環境構築とコアロジック)"
date: 2021-07-24T21:44:42+09:00
description: "Flaskの練習として、英単語練習アプリを作ってみました。"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: DogsCox
tags:
- Flask
series:
- "Flask入門"
categories:
- "Webアプリ"
---

[Flask](https://flask.palletsprojects.com/en/2.0.x/ "flask") を使用して、英単語を覚えるための簡単な英単語学習用アプリを作成する過程を説明します。  
完全に練習目的で作成するアプリです(^^;)  

<!-- START MoshimoAffiliateEasyLink -->
<script type="text/javascript">
(function(b,c,f,g,a,d,e){b.MoshimoAffiliateObject=a;
b[a]=b[a]||function(){arguments.currentScript=c.currentScript
||c.scripts[c.scripts.length-2];(b[a].q=b[a].q||[]).push(arguments)};
c.getElementById(a)||(d=c.createElement(f),d.src=g,
d.id=a,e=c.getElementsByTagName("body")[0],e.appendChild(d))})
(window,document,"script","//dn.msmstatic.com/site/cardlink/bundle.js?20220329","msmaflink");
msmaflink({"n":"Python FlaskによるWebアプリ開発入門 物体検知アプリ\u0026機械学習APIの作り方【電子書籍】[ 佐藤昌基 ]","b":"","t":"","d":"https:\/\/thumbnail.image.rakuten.co.jp","c_p":"","p":["\/@0_mall\/rakutenkobo-ebooks\/cabinet\/3000\/2000010803000.jpg"],"u":{"u":"https:\/\/item.rakuten.co.jp\/rakutenkobo-ebooks\/73fc9be77e1f31e1a3881917995da2fb\/","t":"rakuten","r_v":""},"v":"2.1","b_l":[{"id":1,"u_tx":"楽天市場で見る","u_bc":"#f76956","u_url":"https:\/\/item.rakuten.co.jp\/rakutenkobo-ebooks\/73fc9be77e1f31e1a3881917995da2fb\/","a_id":2682976,"p_id":54,"pl_id":27059,"pc_id":54,"s_n":"rakuten","u_so":1}],"eid":"JLRkL","s":"s"});
</script>
<div id="msmaflink-JLRkL">リンク</div>
<br>
<!-- MoshimoAffiliateEasyLink END -->

随時更新していきますが、以下のリポジトリで作成していきます。  
現在までできている部分で version 0.1 としてタグ付けしています。  

[flask webapp version 0.1](https://github.com/DogsCox/english_word_practice_flask_webapp/tree/fbad1d9f2399a2718b3be427b0381f8c0f38c491 "v0.1")

今回は第一回目の記事で、環境構築から英単語学習問題を出題するためのコアロジックまでの解説です。  
ウェブアプリ機能については次回の記事で説明するので、Flaskの使い方についてご興味のある方は  
[次回の記事]({{<ref "/posts/english_practice_webapp_flask_app.md" >}}) を参照してください。  


## Flaskとは
[Flask](https://flask.palletsprojects.com/en/2.0.x/ "flask")はPythonでWebアプリを作成するためのフレームワークを提供するライブラリです。  
最低限の機能しか持たないため軽量である一方、カスタマイズ性が高いのも特徴です。いわゆるマイクロフレームワークと呼ばれます。  

Python用のWebフレームワークとしては [Django](https://www.djangoproject.com/ "django") というライブラリもあります。   
DjangoはWebアプリを作成するための機能がモリモリのリッチなライブラリです。いわゆるフルスタックライブラリと呼ばれます。

Flaskはプロトタイプや比較的小さなサービス、APIを作成するのに向いていて、  
Djangoはある程度大きなサービスを展開するのに向いていると言えます。  


## 今回作成するアプリ
今回作成するのは日本語に対応した英単語を答える問題が3問出題され、  
回答すると正解と正解数を表示する簡単なウェブアプリです。  

以下の画像のように日本語と英単語の品詞が出題され、空欄に英単語を入力すると...  
![problems](/images/english_practice_webapp_flask_core/problems.png "problems")  

以下のように正解、入力した回答、正解数を表示します。  
![answers](/images/english_practice_webapp_flask_core/answer.png "answers")  

今の所htmlやcssを全然作り込んでいないため、ものすごいシンプルな（というか手抜きの）画面になっています...  
今後画面のデザインの方も頑張りたいと思います。  


## 環境構築
ウェブアプリを作成するための環境を構築していきます。  
こちらも勉強のためにdocker, docker-composeで環境を作っていきます。  
今後DBとの連携もしたいので、docker-composeを使ってみました（今のところはアプリ用のコンテナだけですが）。  

Pythonの公式イメージをベースに、Flaskをpipでインストールしてイメージを作成します。  


### requirements.txt
まずはPythonで `pip install` するライブラリを `/requirements.txt` に記述していきます。  

```
# /requirements.txt
flask
```

今はFlaskだけですが、今後ログイン機能やDB連携機能のためのライブラリを追加していきます。  


### Dockerfile
次にイメージを作成する手順を `/Dockerfile` に書いていきます。  

``` Dockerfile
# /Dockerfile
FROM python:3.9.6-buster

# パッケージインストールとlocaleの設定
RUN apt-get update && \
    apt-get -y upgrade && \
    apt-get -y install sudo curl wget less vim locales && \
    localedef -f UTF-8 -i ja_JP ja_JP.UTF_8
ENV LANG ja_JP.UTF-8
ENV LANGUAGE ja_JP:ja
ENV LC_ALL ja_JP.UTF-8
ENV TZ JST-9

# キャッシュクリア
RUN apt-get clean

# pythonライブラリインストール
RUN python -m pip install --upgrade pip
RUN python -m pip install --upgrade setuptools
COPY requirements.txt /tmp/
RUN python -m pip install -r /tmp/requirements.txt

# ユーザーを作成
ARG DOCKER_UID=xxx
ARG DOCKER_USER=dogscox
ARG DOCKER_PASSWORD=zzz
## 作成したユーザをsudoグループに加える
RUN useradd -m --uid ${DOCKER_UID} --groups sudo ${DOCKER_USER} \
  && echo ${DOCKER_USER}:${DOCKER_PASSWORD} | chpasswd

## 作成したユーザーに切り替える
USER ${DOCKER_USER}
# ワーキングディレクトリ変更
WORKDIR /home/dogscox
```

Dockerfileの `# ユーザーを作成` の部分でコンテナ内外のUIDを同じに設定しています。  
(`xxx`にローカルで`id`コマンドで調べたUIDを入力します。)  
コンテナはデフォルトだとrootユーザでログインするので、このように設定しないと  
権限の問題からコンテナで編集したファイルをコンテナ外では編集できなくなってしまいます。  

これがベストプラクティスなのかどうかはわからないのですが...  

コンテナ内でssh鍵を作成してソースをgit管理しても良いのですが、  
自分はVScodeの[Remote Container拡張機能](https://zenn.dev/suirunakamura/articles/3a83bdf6847970 "remote_container")でコンテナ内のファイルを編集し、  
コンテナ外でファイルをgit管理しています。  
(開発用のコンテナとデプロイ用のコンテナを分けて作成し、開発用コンテナ内でソースをgit管理するのが良いかも？)  


### docker-compose.yml
最後にコンテナを起動する際の設定を `/docker-compose.yml` に書いていきます。  
今回はコンテナを一つしか作成していないのでdocker-composeを使う必要はないのですが、  
今後データを格納するデータベースもコンテナで作成する予定なので、今のうちにdocker-composeで環境を作っておきます。  

``` yml
# /docker-compose.yml
version: '3'
services:
    app:
        restart: always
        build: .
        container_name: 'app'
        working_dir: '/home/dogscox'
        tty: true
        ports:
            - 8080:5000
        volumes:
            - .:/home/dogscox
```

実は `docker-compose.yml` を書いたのはこれが初めてなので、正しく書けているかはわかりません...  
`working_dir`をDockerfileの`WORK_DIR`に合わせています。(不要かも？)  
また、コンテナのポート5000をローカルのポート8080にフォワーディングしています。  

Flaskでウェブアプリを立ち上げた際にデフォルトでポート5000を開くので、このようにしています。  

ここまでで環境構築は終わりです。  
以下のコマンドでコンテナをbuildし、コンテナを起動します。  

``` bash
$ docker-compose build
$ docker-compose up
```

あとはVScodeからコンテナに接続し、ソースを編集します。  
( [Remote Container拡張機能](https://zenn.dev/suirunakamura/articles/3a83bdf6847970 "remote_container") )  

コンテナを落とす際は以下のようにします。  

``` bash
$ docker-compose down
```


## ディレクトリ構成
まず、全体像として今回作成するウェブアプリのディレクトリ構造を示します。  
今回作成するのはデータを格納する `/data/` 以下とロジック部分の `/eng_practice/` 以下となります。   

```
<root>
|
|--Dockerfile           # 環境構築用
|--docker-compose.yml   # 環境構築用
|--requirements.txt     # 環境構築用
|--README.md
|--app.py               # Flaskのviewを定義したファイル。アプリ本体
|
|--data                 
|  |--english_words.csv # 英単語と日本語訳を格納したファイル
|  |--id_pass.txt       # 認証用
|
|--eng_practice
|  |--__init__.py       # 空ファイル
|  |--eng_question.py   # 英単語学習用の問題を出題する機能
|
|--templates
|  |--eng_prac.html     # 英単語学習問題を出題する画面のtemplate
|  |--index.html        # メニュー画面のtemplate
|  |--login.html        # ログイン画面のtemplate
|  |--result.html       # 回答結果画面のtemplate
```


## コアロジック構築
それでは、英単語問題出題のためのコードを書いていきます。  


### 英単語データ作成
まず出題元となる英単語データを作成します。  
今回は出題元となる英単語のリストをCSVファイルにベタ書きしていきます。  
(今後きちんとデータベースに移行します...)  

``` csv
/data/english_words.csv
1,cat,ねこ,名詞
2,beautiful,うつくしい,形容詞
3,they,彼ら、彼女ら,代名詞
4,cute,かわいい,形容詞
5,run,走る,動詞
6,eat,食べる,動詞
7,cool,冷たい,形容詞
8,hot,暑い,形容詞
9,read,読む,動詞
10,head,頭,名詞
```

...はい。  
今回出題できるのはこの10単語だけです...

1列目が単語のID、2列目が答えとなる英単語、3列目が英単語に対応する日本語、4列目が英単語の品詞です。  


### ログイン認証用のファイル作成
次に、一応練習としてIDとパスワードでの認証を実装してみます。  
今後きちんと実装しますが、今のところはIDとパスワードをファイルにベタ書きしてそれとの一致で認証します。  

```/data/id_pass.txt
John,Due
```

1列目がID、2列目がパスワードのつもりです。  


### 英単語練習問題を出題する機能
今回のコアロジックとなる英単語学習問題を出題する機能を実装します。  
コアロジックと言っても`/data/english_words.csv`を読み込み、その中からランダムに3つ選択するだけです。  

``` python
# /eng_practice/eng_question.py
from random import sample

def send_eng_ja_pair(path):
    problems = []

    with open(path, 'r', encoding='utf-8') as f:
        for row in f:
            id, eng, ja, tp = row.rstrip().split(',')

            # 単語と意味の辞書をリストに格納
            problems.append({
                'id': id,
                'word': eng,
                'ja': ja,
                'type': tp
            })

    # ランダムに3つ選択
    problems_sampled = sample(problems, 3)
    # ランダムに選んだ順に順序をつける
    for i, p in enumerate(problems_sampled):
        p['ord'] = i

    return problems_sampled


# テスト用コード
if __name__ == '__main__':
    p = send_eng_ja_pair('../data/english_words.csv')
    print(p)
```

途中、以下のように選んだ順序で番号をつけています。   
これはwebページとやり取りする際に確実に順序を再現するためです。  

``` python
# ランダムに選んだ順に順序をつける
for i, p in enumerate(problems_sampled):
    p['ord'] = i
```

それでは `eng_question.py` をテスト実行してみましょう。  

```bash
$ python eng_question.py
[{'id': '6', 'word': 'eat', 'ja': '食べる', 'type': '動詞', 'ord': 0}, {'id': '9', 'word': 'read', 'ja': '読む', 'type': '動詞', 'ord': 1}, {'id': '4', 'word': 'cute', 'ja': 'かわいい', 'type': '形容詞', 'ord': 2}]
```

想定通り英単語の情報と順序 (`'ord'`) がきちんと返ってきていますね。  

今回説明するのはここまでです。  
[次回の記事]({{<ref "/posts/english_practice_webapp_flask_app.md" >}})ではランダムに選んだ英単語の情報を用い、英単語練習アプリのアプリ部分を作っていきます。  
