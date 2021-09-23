---
title: "【Flask入門 2.5】Flask で作る英単語学習アプリ v0.2 (リファクタリング編)"
date: 2021-09-21T14:53:56+09:00
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: DogsCox
tags:
- "Python"
- "Flask"
series:
- "Flask入門"
categories:
- "Webアプリ"
---

この記事では、 [前回の記事]({{<ref "/posts/english_practice_webapp_flask_app.md" >}}) までで作成した英単語学習アプリのコードを「MTVフレームワーク」に沿ってリファクタリングしていきます。  

リファクタリング後のコードは以下です。  
[英単語学習アプリv0.2](https://github.com/DogsCox/english_word_practice_flask_webapp/tree/010a13c7810540636661557a3b3590b52f800d5e "v0.2")  

前回までの記事で、「とりあえずモノが動く」というところまで辿り着きました。  
しかし、アプリの本体である [app.py](https://github.com/DogsCox/english_word_practice_flask_webapp/blob/fbad1d9f2399a2718b3be427b0381f8c0f38c491/app.py "app.py") にほぼウェブアプリとしての全機能が詰め込まれている状態です。  
この `app.py` を機能ごとにファイルを分割して今後の機能拡張やコードの修正をやりやすくします。  

ファイルに様々な機能が集約されているという状況は、いわば服や雑貨など色々な物をクローゼットの一つの棚に入れている状態です。  
（個人的なイメージですが。）  
その中にはよく使うもの、そんなに使わないもの、時々は確認しないといけない重要なものなどが入っているでしょう。  
このような状態だと後々整理するの大変ですよね。  
また、新しい物をどんどん追加したりすると前から入っていたものを忘れてしまいそうです。  

なので、「これはこの棚、あれはあの棚」のように分けて収納して管理しましょう。  
プログラムをファイルやディレクトリに分けて格納するのも似たような理由です。  

さて、それではどのように分けるのが良いでしょうか？  
今は Flask を使用しているので、 Flask が推奨する方法があればそちらに従うのが良さそうです。  

Flask は「MTV フレームワーク」を採用しています。  
そのため、整理の仕方もこの MTV フレームワークに従います。  

以下では MTV フレームワークについて簡単に説明したあとに実際にコードをリファクタリングしていきます。  

余談ですが、[cookiecutter-flask](https://github.com/cookiecutter-flask/cookiecutter-flask "cc_flask") をダウンロードすることで簡単にベストプラクティスに近いディレクトリ、ファイル構造を実現することが可能です。  
[cookiecutter](https://cookiecutter.readthedocs.io/en/1.7.2/ "cc") は様々なプロジェクトのディレクトリ、ファイル構造の雛形を提供してくれます。  
雛形が用意されているならこちらを利用するのが楽です。  

## Flask の MTV フレームワーク
まず、ウェブアプリの構成要素について考えてみます。  
ウェブアプリが世に現れて数十年、既に基本の構成は固まっています。  
基本的に、必要なのはサーバサイドのアプリケーション、データベース、画面制御のフロントエンドの3つです。  
それぞれの役割を以下に示します。  

1. サーバサイドのアプリケーション
    - ユーザとやり取りして必要な情報を特定の形に整理して画面を制御するプログラムに渡します。
1. データベース
    - ユーザが必要とする情報をためておき、サーバサイドのアプリケーションの要請に応じて情報を返します。
1. 画面制御のフロントエンド
    - 画面に特定の形で情報を表示します。
    - 基本的に HTML, CSS, JavaScript で構成されます。

Flask の MTV テンプレートとは、

- サーバサイドのアプリケーション <----> V: View
- データベース <----> M: Model
- フロントエンド <----> T: Template

と対応付けてプログラムを管理する方法です。  
以下、それぞれについてもう少し詳しく説明します。  

### T: Template
Template はフロントエンド、つまり画面に何を表示するかを定める部分です。  
主に HTML で構成され、必要に応じて CSS で見栄えを良くして JavaScript で動きを付けます。  

Flask の場合、Template には View を通してデータが渡されます。  
また、 Flask では [Jinja2テンプレート](https://jinja.palletsprojects.com/en/3.0.x/ "jinja2") を利用して画面に表示するコンポーネントをプログラムで制御することができます。  


### M: Model
Model はデータベースとのやり取りを担当する部分です。  
View を通して伝えられるユーザの要求に沿ったデータをデータベースから抽出します。  

データベースを操作するには普通は SQL という独自の言語を使用する必要があるのですが、  
Model を定義することで Model に対して必要なデータを指示するだけでデータを取ってこられるようにします。  
このような仕組みを Object Relation Mapper (ORマッパー) と呼びます。

基本的には `Flask-SQLAlchemy` というライブラリを用います。  

今の所作成している英単語学習アプリはデータベースと連携していないので、 Model は今回は作成しません。  
今後機能拡張していきます。  


### V: View
これまでの記述でなんとなくわかるかもしれませんが、  
View が Flask の場合ウェブアプリの司令塔です。  

ユーザから情報を受けとり、どのような情報が必要かを判断して Model を通してデータを受け取ります。  
そして Template に受け取ったデータを加工して渡す役割を果たします。  

さらに、 View はどの処理にどの Template (つまり HTML) を紐付けるかを定義し、  
処理に応じた画面の遷移を定める役割も持ちます。  

以上が大まかな Flask の MTV フレームワークの考え方です。  
以降ではこの MTV フレームワークに沿うように英単語学習アプリを改修していきます。  


## 英単語学習アプリのリファクタリング
それでは MTV フレームワークに沿ってコードをリファクタリングしていきます。  

なお、ディレクトリ構成などは [Flaskのチュートリアル](https://study-flask.readthedocs.io/ja/latest/02.html "flask_tutorial") を参考にさせていただきました。  

まずは現状のディレクトリ構成を示します。  

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

`app.py` の概略を示します。

``` Python
from flask import Flask, render_template, session, request, redirect, url_for
...

# インスタンスの生成
app = Flask(__name__)

# セッションで使う鍵を設定
key = os.urandom(21)
app.secret_key = key

# 各々のページのViewを記述
# ランディングページ
@app.route("/")
def index():

...

# アプリケーションの起動
if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

まず気になるのは、 `app.py` において View の記述部分と暗号鍵の作成などのアプリケーションの設定、起動部分が同じファイルに記述されている点です。  
これらは機能として全く別物なので、分離してしまったほうが良いでしょう。  

また、上記の分離を行ったとして、 View 部分のみを格納するディレクトリ、ファイルがありません。  
まずはアプリの起動スクリプトとアプリ本体を切り離し、アプリ本体用のディレクトリ内で MTV それぞれに対応するディレクトリ、コードを作成するようにします。  

### アプリ起動スクリプトの分離
まずはウェブアプリ起動スクリプトを `app.py` から独立させましょう。  
アプリの起動は `app.py` の最後の部分なので、これを別ファイル (`manage.py`) に書き出します。  

``` Python
# /manage.py
from eng_prac_app import app


# アプリケーションの起動
if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

ライブラリをインポートしている部分の `eng_prac_app` はこれから作成するウェブアプリ本体のディレクトリです。  
`manage.py` はそちらの `app` をインポートし、起動するだけの役割を担ってもらいます。  

ちなみにアプリ起動用のコマンドは  

``` bash
$ python manage.py
```

となります。  


### ウェブアプリ本体ディレクトリの作成
次に、ウェブアプリの本体 (MTV それぞれのコード) を一つのディレクトリにまとめてしまいます。  
ディレクトリの名前は `eng_prac_app` としておきます。  

`eng_prac_app` の中にアプリを構成するコードをまとめてしまいます。  
`eng_prac_app` 内の構成は以下のようにします。  

```
</eng_prac_app/>
|--__init__.py
|--scripts
|  |--__init__.py
|  |--eng_question.py
|--templates
|  |--eng_prac.html
|  |--index.html
|  |--login.html
|  |--result.html
|--views.py
```

いきなり今まで存在していなかったディレクトリ、ファイルが出てきたと思うので一つづつ説明します。  

- `scripts/` ディレクトリ
    - 前回作成した `eng_practice` ディレク卜リです。英単語出題用の関数を格納していました。
    - 今後も何かしら必要な補助関数などを作成することを考えて `scripts` という名前にしました。
    - 名前を変えて移動しただけです。
- `templates/` ディレクトリ
    - こちらは前回作成した `templates` ディレクトリを移動しただけです。
- `eng_prac_app/__init__.py`
    - アプリの設定部分のみを記述したライブラリ設定用のファイルです。後で説明します。
- `views.py`
    - もとの `app.py` で View を定義した部分を記述したファイルです。後で説明します。

では、各々の中身について詳細を説明します。  


### `eng_prac_app/__init__.py`
`app.py` で行っていたアプリのインスタンスの生成と設定の部分のみを行うコードです。  

``` Python
from flask import Flask
import os

# インスタンスの生成
app = Flask(__name__)

# セッションで使う鍵を設定
key = os.urandom(21)
app.secret_key = key

import eng_prac_app.views
```

`__init__.py` が何をしているのかについては様々な記事で説明されているのでそちらをご参考にしてください。  

- [Python の __init__.py とは何なのか](https://qiita.com/msi/items/d91ea3900373ff8b09d7 "init1")
- [[Python] __init__.pyの役割とは？パッケージ化やimportの関係は？](https://shimi-dai.com/python-what-is-__init__-py/ "init2")


### `eng_prac_app/views.py`
`app.py` で定義した View のみを切り出したファイルになります。  

``` Python
from flask import Flask, render_template, session, request, redirect, url_for
from pathlib import Path
import os
from eng_prac_app import app
from eng_prac_app.scripts.eng_question import send_eng_ja_pair


# ランディングページ
@app.route("/")
def index():
    if not session.get("login"):
        return redirect(url_for("login"))
    else:
        return render_template("index.html")

@app.route...

```

Model 部分はまだデータベース連携機能が無いので作成していませんが、  
作るとしたら `eng_prac_app/models.py` といったファイルに作成することになります。  


### リファクタリング後のディレクトリ構成
以上のリファクタリングを行った後の全体のディレクトリ構成を示します。  

```
|--Dockerfile            # 環境構築用
|--requirements.txt      # 環境構築用
|--docker-compose.yml    # 環境構築用
|--README.md 
|--data                  # 英単語データ格納ディレクトリ
|  |--english_words.csv
|  |--id_pass.txt
|--eng_prac_app          # ウェブアプリ本体
|  |--__init__.py
|  |--scripts
|  |  |--__init__.py
|  |  |--eng_question.py
|  |--templates
|  |  |--eng_prac.html
|  |  |--index.html
|  |  |--login.html
|  |  |--result.html
|  |--views.py
|--manage.py             # アプリ起動用ファイル
```

トップ階層で、

``` bash
$ python manage.py
```

と打つとアプリが起動します。アプリの動作は前回と変更ありません。  

再掲ですが、リファクタリング後のコードの全体は github のリポジトリにもアップしてあります。  
[英単語学習アプリv0.2](https://github.com/DogsCox/english_word_practice_flask_webapp/tree/010a13c7810540636661557a3b3590b52f800d5e "v0.2")  


## まとめ
はい、ということで気になっていた部分のリファクタリングをしてみました。  
このような記事にどれほど需要があるのかは謎ですが...
今後アプリのアップデートをして再度ブログ記事を投稿したときに、  
「あれ？前回と全然違わないか？」  
とならないためにリファクタリングの過程も記事にしました。  

次はいい加減データベースを作成してデータベース連携機能を作りたいですね。  
今後もアプリのアップデートに合わせて随時記事を投下していくつもりなので、その際も読んで頂けると幸いです。  
