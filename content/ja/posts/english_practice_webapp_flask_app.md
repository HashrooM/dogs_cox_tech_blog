---
title: "【Flask入門 2】Flaskで作る英単語学習アプリver0.1(アプリ構築)"
date: 2021-07-31T16:05:48+09:00
description:
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

[前回の記事]({{<ref "/posts/english_practice_webapp_flask_core.md" >}})に引き続き、英単語学習用アプリを作成していきます。  
前回の記事で環境構築と練習問題を出題するためのロジックを実装するところまで作成したので、  
今回の記事では実際にウェブページとして表示するためのコードを作成していきます。  

ウェブアプリにおけるルーティングとはなにか、Flaskでどのようにウェブアプリを作っていくのかについてはここでは述べません。  
公式のチュートリアルや様々な書籍、ブログ記事で紹介されているのでそちらをご覧ください。  
(気が向いたら練習がてらこのブログでも解説記事を書くかもしれません。)  

- [公式チュートリアル](https://flask.palletsprojects.com/en/2.0.x/tutorial/index.html "tutorial")
- [Webアプリ初心者のFlaskチュートリアル](https://qiita.com/kiyokiyo_kzsby/items/0184973e9de0ea9011ed "qiita")
- [Python & Flask 初めてのWebアプリケーション作成に挑戦!!(amazonへのリンク)](https://www.amazon.co.jp/Python-Flask-%E5%88%9D%E3%82%81%E3%81%A6%E3%81%AEWeb%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E4%BD%9C%E6%88%90%E3%81%AB%E6%8C%91%E6%88%A6-%E9%AB%99%E9%A0%AD%E5%91%A8%E5%B9%B3-ebook/dp/B08M6979ZS/ref=sr_1_2?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&dchild=1&keywords=flask+web%E3%82%A2%E3%83%97%E3%83%AA&qid=1628000116&sr=8-2)
    - 本記事の内容はこちらの書籍を参考にさせていただきました。


## 作成するウェブアプリ
前回の記事でも言及したように、日本語と品詞に対応する英単語を入力すると  
正解と正答数を表示する簡単なものです。  

問題回答画面
![problems](/images/english_practice_webapp_flask_core/problems.png "problems")  

正解確認画面  
![answers](/images/english_practice_webapp_flask_core/answer.png "answers")  


## ページ遷移
作成するページとページ間の遷移は以下のようにします。  
![pagenation](/images/english_practice_webapp_flask_app/pagenation.png "pagenation")  


## ディレクトリ構成
コードはgithubで公開しています。  
[flask webapp version 0.1](https://github.com/DogsCox/english_word_practice_flask_webapp/tree/fbad1d9f2399a2718b3be427b0381f8c0f38c491 "v0.1")

[前回の記事]({{<ref "/posts/english_practice_webapp_flask_core.md" >}})でも書きましたが、今回作成したディレクトリ構成を以下に示します。  
今回作成するのはルーティング処理を実現する `/app.py` と表示するウェブページ本体である `/templates/` 以下です。  

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


## ウェブアプリ作成
それでは、実際にアプリとして必要になる処理と表示するhtmlファイルを作成していきます。  


### Flaskを利用する準備
`/app.py`にルーティング処理を記述していきます。  
そのためにまずは`/app.py`ファイルを作成し、ライブラリのインポートなどの準備を記述します。  
また、[前回の記事]({{<ref "/posts/english_practice_webapp_flask_core.md" >}})で作成した英単語練習問題を出題するための関数 `send_eng_ja_pair` もimportしておきます。  

``` python
# /app.py
from flask import Flask, render_template, session, request, redirect, url_for
from pathlib import Path
import os
from eng_practice.eng_question import send_eng_ja_pair

# インスタンスの生成
app = Flask(__name__)

# セッションで使う鍵を設定
key = os.urandom(21)
app.secret_key = key
```

今回はログイン情報や出題した問題の情報などをセッション情報として書き込むため、  
セッションを扱えるようにしています。

次に、実際にwebページに表示するhtmlファイルは`/templates/`以下に作成していきます。  
そのために`/templates`ディレクトリを作成しておきましょう。  

### メニュー画面
まず、サイト訪問者が一番最初にたどり着くページ、ランディングページ(LP)にやってきたときの処理とLPのhtmlファイルを作成します。  

LPにユーザが訪問した際は、ユーザがログイン済みかどうかをチェックし、  
もしログインしていなければログインページへ、ログインしていればメニュー画面を表示します。  

この処理をFlaskで書いたコードが以下になります。  

``` python
# /app.py
# ランディングページ
@app.route("/")
def index():
    if not session.get("login"):
        return redirect(url_for("login"))
    else:
        return render_template("index.html")
```

訪問したユーザがログインしているか否かをセッション情報から取り出しています。   

ログイン画面については後で作成することとし、まずはメニュー画面を作成します。  
メニュー画面は `index.html` というファイル名で作成します。  
(`index.html`はFlaskがデフォルトで表示するhtmlファイルになります。)  

``` html
<!-- /templates/index.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>index</title>
</head>
<body>
    <h2>英単語練習アプリ</h2>
    <a href="/eng_prac">英単語の練習をする</a>
    <br>
    <a href="/logout">ログアウト</a>
</body>
</html>
```

実際に表示される画面は次のようになります。  
![index](/images/english_practice_webapp_flask_app/index.png "index")  

ちなみに豆知識ですが、エディタとしてVScodeを使用している場合、  
拡張子`.html`のファイルで`html5`と入力してtabを押下すると、以下のようにhtmlファイルのテンプレートが展開されます。  
(Emmetというデフォルトで使用できるプラグインの機能です。表示内容はVScodeのバージョンに依存すると思います。)    

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>index</title>
</head>
<body>

</body>
</html>
```

デフォルトだと `<html lang="en">` と英語が指定されているのですが、  
VScodeの `setting.json` で以下のようにデフォルトで日本語になるように設定変更しておくと便利です。  

``` json
    "emmet.variables": {"lang" : "ja"}
```

参考：[【VSCodeでEmmet入門】 Emmetを使ってHTMLコーディングを効率化しよう。](https://technical-creator.com/vscode-emmet/ "emmet")  


### ログイン、ログアウト
次にログイン画面を作成します。  
最初に訪問したユーザが到達するのは実際はこのログイン画面となります。  

ログイン画面ではIDとパスワードによる認証を求め、認証が通ったら前節のメニュー画面に遷移します。  
まずはログイン画面における処理を実装します。  
ログイン画面ではログインフォームを表示するのみです。  

``` python
# /app.py
# ログインページ
@app.route("/login")
def login():
    return render_template("login.html")
```

ログイン画面に表示するhtmlファイルを作成します。  

``` html
<!-- /templates/login.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>login</title>
</head>
<body>
   <h2>ユーザIDとパスワードを入力してください</h2> 
   <form method="POST" action="/logincheck">
        <p>ユーザID</p>
        <input placeholder="ユーザID" type="text" name="user_id">
        <p>パスワード</p>
        <input placeholder="パスワード" type="text" name="password">
        <br>
        <input type="submit" value="送信">
    </form>
</body>
</html>
```

htmlではシンプルにログインIDとパスワードの入力フォームを表示し、内容をPOSTメソッドで送信するだけです。  
送信先を `/logincheck` ページとしています。  
`/logincheck`ページではIDとパスワードが事前に登録されたものかどうかをチェックします。  
`/logincheck`ページは認証だけなので、表示するhtmlファイルはありません。処理のみ記述します。  

``` python
# /app.py
# ログイン認証
@app.route("/logincheck", methods=["POST"])
def logincheck():
    user_id = request.form["user_id"]
    password = request.form["password"]

    # /data/id_pass.txtからIDとパスワードを読み込む
    valid_users = {}
    with open("./data/id_pass.txt", 'r', encoding="utf-8") as f:
        for row in f:
            id, pw = row.rstrip().split(",")
            valid_users[id] = pw

    # 認証
    if user_id in valid_users.keys():
        if password == valid_users[id]:
            session["login"] = True
        else:
            session["login"] = False
    else:
        session["login"] = False

    # 認証の状態に応じてリダイレクト
    if session["login"]:
        return redirect(url_for("index"))
    else:
        return redirect(url_for("login"))
```

`/login`ページから送信されたIDとパスワードを受け取り、`/data/id_pass.txt`に事前に記述したID、パスワードと一致するかチェックします。  
一致しなければ再度ログインページを表示し、一致すればセッションにログイン情報を書き込み、メニュー画面に遷移します。  

ちなみに今はID、パスワードをファイルにベタ書きしていますが、本来は暗号化してDB等に格納すべきかと思います。  
(`flask_login`ライブラリを使えばそれほど手間をかけずに実装可能です。なのでいずれ対応します。いずれ...)  
ID、パスワードは以下のようにカンマ区切りでファイルに書いておきます。  

``` /data/id_pass.txt
John,Due
```

`ID, パスワード` の順です。  

ついでにログアウト処理も書いておきましょう。  
ログアウトも表示するページはないので処理のみ記述します。  

``` python
# /app.py
# ログアウト
@app.route("/logout")
def logout():
    session.pop("login", None)
    return redirect(url_for("index"))
```

ログアウトした際はセッション情報を削除し、メニュー画面に遷移します。  
とはいえログアウト状態でメニュー画面に移動した場合はログイン画面にリダイレクトするので、  
実際はログイン画面に遷移します。  


### 英単語の学習問題出題画面
では、英単語の学習問題をユーザに出題し、回答してもらう画面と処理を書いていきます。  

まずは処理から書いていきましょう。  

``` python
# /app.py
# 英単語練習回答ページ
@app.route("/eng_prac")
def eng_prac():
    # 英単語辞書ファイルへのpath
    path = Path()
    path = path.cwd() / "data" / "english_words.csv"
    path = path.resolve()
    probs = send_eng_ja_pair(path)
    session["probs"] = probs
    return render_template("eng_prac.html", problems=probs)
```

英単語練習回答ページではファイルに格納した英単語情報から3つ選んで問題を出題する`send_eng_ja_pair`関数を呼び出しています。  
`send_eng_ja_pair`関数は必要な情報を辞書として返します。  
この辞書をセッション情報に書き込み、セッションを通して情報を保持するようにしています。  

後で回答チェックの際にこのセッション情報から正解となる情報を取得するのですが、  
その際に辞書の順序が保存されていないようだったので`send_eng_ja_pair`関数では順序を保持するための情報も付け加えています。  
([前回の記事]({{<ref "/posts/english_practice_webapp_flask_core.md" >}})参照)  

では、出題画面のhtmlファイルを作成していきます。  


``` html
<!-- /templates/eng_prac.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>eng_prac</title>
</head>
<body>
    <h2>日本語と品詞に対応する英単語を入力してください。</h2>
    <form method="POST" action="/anscheck">
        {% for prob in problems %}
        <p>{{ prob['ja']  }}【{{ prob['type']}}】</p>
        <input type="text" name="words" size="15">
        {% endfor %}
        <br>
        <input type="submit" value="送信">
        <br>
        <a href="/">メニューへ</a>
        <br>
        <a href="/logout">ログアウト</a>
    </form>
</body>
</html>
```

日本語と英単語の品詞を表示し、回答となる英単語を入力するためのフォームを表示しています。  
入力内容はPOSTメソッドで正解チェック用のページである`/anscheck`ページに送信します。  

練習問題は3問出題するので、Flaskで使用することのできるjinja2テンプレートのfor文を使用して  
3つのフォームを作成しています。  

jinja2については以下のような記事が参考になると思います。  

- 公式ページ：[jinja](https://jinja.palletsprojects.com/en/3.0.x/ "jinja")
- [Jinja2の使い方を紹介。Flask＋Jinja2](https://www.sukerou.com/2019/04/jinja2flaskjinja2.html "jinja_blog")

for文で作成されるフォームの全てで`name="words"`を指定しています。  
このようにしておくと、Flaskの場合は`words`情報を受け取る際にリストとして受け取ることができます。  

回答画面は以下のようになります。  
![problems](/images/english_practice_webapp_flask_core/problems.png "problems")  



### 正解チェック画面
最後に正解チェック画面です。  
回答画面から送られてきた回答とセッション情報として保持していた正解を突き合わせ、  
正解の数を表示します。  
合わせて回答と正解も表示します。  

``` python
# /app.py
# 英単語練習正解チェックページ
@app.route("/anscheck", methods=["POST"])
def anscheck():
    user_ans = request.form.getlist("words")
    probs = session.get("probs")
    # 出題順序と同じになるようにsort
    probs = sorted(probs, key=lambda x: x['ord'])

    # ユーザの回答結果
    result = []
    # ユーザの正解数
    correct_num = 0
    for i, w in enumerate(user_ans):
        if w == probs[i]["word"]:
            result += "正解"
            correct_num += 1
        else:
            result += "不正解"
    return render_template(
        "result.html",
        correct=correct_num,
        ans=zip(probs, user_ans)
    )
```

セッション情報から出題した問題の情報を取り出し、  
正解数を数えた上で表示のために情報を`/anscheck`ページへ渡しています。  

セッションから情報を取り出したあとも辞書の順序を保持するために順序のキーでソートし直しています。  

では、`/ansckeck`画面で表示する`result.html`ファイルを作成していきましょう。  

``` html
<!-- /templates/result.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>result</title>
</head>
<body>
    <h2>回答</h2>
    {% for c, a in ans %}
    <p>{{ c['ja']  }}【{{ c['type'] }}】の答えは... <strong>{{ c['word'] }}</strong> あなたの答えは ... {{ a }}</p>
    {% endfor %}
    <br>
    <p>あなたの正解数は ... <strong>{{ correct }} / 3</strong> でした。</p>
    <br>
    <a href="/eng_prac">もう一度やる</a>
    <br>
    <a href="/">メニューへ</a>
    <br>
    <a href="/logout">ログアウト</a>
</body>
</html>
```

ここでもjinja2テンプレートを用いて渡された辞書の情報をfor文で書き出しています。  

正解チェック画面は以下のようになります。  
![answers](/images/english_practice_webapp_flask_core/answer.png "answers")  


### アプリ起動
最後にアプリを起動するコードを付け足して完成です。  

``` python
# /app.py
# アプリケーションの起動
if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

引数で`host=0.0.0.0`としていますが、これは「どのIPアドレスからのアクセスも受け付ける」という意味になります。  
Dockerコンテナ上で開発しているため、コンテナのlocalhostとローカルのlocalhostが異なるものになり、このようにする必要が出てきます。  
(当然アプリ公開時にも必要です。)  

Flaskはデフォルトでは外部に公開しない設定にしているらしく、公開するのであればこのようにする必要があるとのことでした。  
地味に詰まりました。  
[Flaskのサーバーはデフォルトだと公開されてない](https://qiita.com/tomboyboy/items/122dfdb41188176e45b5 "flask_host")

`port=5000`はデフォルト値なので指定する必要はないのですが、後々書き換えることもあるかと思って一応書いておきました。  


## 最後に
お疲れ様でした！！  
これでアプリは完成です！！  

アプリとしては相当シンプルなものですが、  
Flaskを使ったウェブアプリ作成の練習としては十分なボリュームなのではないでしょうか？  

とはいえ画面の作り込みや色々イケてないところもあるので、  
そのあたりは今後随時アップデートさせていきたいです。  
具体的には以下のようなところを作っていきたいところです。  

- 英単語の情報をデータベースに格納
    - 現在出題元となる英単語の一覧をファイルに格納していますが、データベースに格納して管理するよう変更する。  
- 英単語の追加、削除機能
    - 一般的なCRUD処理として、ユーザが勉強したい英単語を新規登録、修正、削除する機能を開発する。
- ログイン機能の実装
    - ファイルにベタ書き＆if文で認証しているのを`flask-login`を用いてもう少しきちんとした認証に変更する。
- サイトのデザイン
    - 見た目があまりにもアレなので、ちゃんと画面をデザインしてhtml, cssをコーディングする。
- herokuへデプロイ
    - やっぱり作ったら公開したくなる。

今後もアップデートを続けていく所存なので、上記のような所を作り次第、記事をアップしていきます。  
