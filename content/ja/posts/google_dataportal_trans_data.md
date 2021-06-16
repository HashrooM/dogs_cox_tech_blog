---
title: "【Googleデータポータル入門：番外編】データを可視化に向いた形式に変換する"
date: 2021-05-26T23:36:38+09:00
description: "可視化を簡単にできるようにPythonでデータを整形します。"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: DogsCox
tags:
- "BI tools"
- "Google Dataportal"
series:
- "How to use Google Dataportal"
categories:
- "Data Analysis"
---

[Googleデータポータル入門：第1回]({{<ref "/posts/google_dataportal_setup.md" >}})と[Googleデータポータル入門：第2回]({{<ref "/posts/google_dataportal_analysis.md" >}})でデータポータルへのデータの連携と簡単な分析について説明しました。  
その際、同じような可視化の手順を繰り返してグラフを作成しました。  
今回の記事では元になるデータの形式を変形することでこのような繰り返しの手間を省く方法を説明します。  
今回説明する内容はプログラミング言語でデータ処理を行うのがメインの多少発展的な内容ですが、  
「コンピュータが処理しやすい形でデータを保存する」という汎用性の高い内容でもあります。  
ぜひ目を通していっていただければと思います。  


## データの形式
[前回の記事]({{<ref "/posts/google_dataportal_analysis.md" >}})で動画視聴サービスごとに棒グラフを作成する際、各視聴サービスごとに同じ操作（指標の作成、フィルタの適用、グラフの作成）を繰り返してグラフを作成しました。  
このように同じ作業を繰り返すのは単純にめんどくさいだけでなく、思わぬミスが生じる可能性があるのでなるべく避けたいところです。  
では、なぜ同じ作業を繰り返す羽目になったのでしょうか？  
それは映画タイトルを視聴できるか否かが __「各視聴サービスごとに列として保存されていた」__ ためです。  
元のデータは以下の図のようになっていました。
![original data](/images/google_dataportal_trans_data/original_data.png "original_data")  

各視聴サービスの列が存在し、各々で視聴可能（1）か視聴不可能（0）かがデータとして表されていますね。  
この形式は人の目では見やすいのですが、コンピュータに処理させようとした際には __「各列ごとに同じ処理を繰り返す」__ という必要があり、ちょっと面倒になります。  
そこで、列ごとに存在している視聴サービスごとの情報を一つの列にまとめてしまうことを考えます。  
イメージとしては、以下のような形式に変形します（というか、この後行う処理で目指す形式がこれです）。
![long data](/images/google_dataportal_trans_data/long_data.png "long_data")  

列として横に伸びたデータを行として縦にまとめてしまうというイメージですね。  
そのためか、元のデータのような列方向に情報が展開されたデータ形式は __横持ち形式__ や __wide format__ 、変換後の行方向に情報がまとめられたデータ形式は __縦持ち形式__ や __long format__ などと呼ばれることもあります。  
縦持ち形式は「コンピュータが処理しやすい整理されたデータ」という意味で、 __tidy data__ と呼ばれることもあります。  
tidy data の考え方はデータを扱う際にとても有用な考え方なので、もしご興味があれば [こちらのサイト](https://id.fnshr.info/2017/01/09/trans-tidy-data/ "tidy_data") から詳細を一読してみてください。  

縦持ち形式のほうがなぜコンピュータで扱いやすいのかは言葉で説明されるだけではちょっとわかりにくいかもしれません。  
こればっかりは「やってみると分かる」としか言えない部分でもあるので、実際にデータを変形してみて、データポータルで可視化してみます。  
（おそらく皆様が使い慣れているエクセルでは横持ち形式のほうが何かと便利だったりします。とはいえ、エクセルは [ピボットデーブル](https://support.microsoft.com/ja-jp/office/%E3%83%94%E3%83%9C%E3%83%83%E3%83%88%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB%E3%82%92%E4%BD%9C%E6%88%90%E3%81%97%E3%81%A6%E3%83%AF%E3%83%BC%E3%82%AF%E3%82%B7%E3%83%BC%E3%83%88-%E3%83%87%E3%83%BC%E3%82%BF%E3%82%92%E5%88%86%E6%9E%90%E3%81%99%E3%82%8B-a9a84538-bfe9-40a9-a8e9-f99134456576 "pivot_table") で簡単に縦持ち→横持ち変換ができるので、データ自体は縦持ちで保存しておき、必要に応じてエクセルで横持ちに変換して可視化、というようなワークフローのほうが様々な状況に対応できると思います。）    


## Pythonで横持ち形式を縦持ち形式に変換する方法
縦持ち→横持ち変換はエクセルやGoogleスプレッドシートのピボットテーブルを使えば簡単にできるのですが、  
横持ち→縦持ち変換は自分が調べてみた限り何らかのプログラミング言語を用いる必要がありそうです。  
そこで今回はデータ分析用のライブラリが揃っているPythonを用いて変換を行います。  
PythonはGoogleが無償で提供している [Colaboratory](https://colab.research.google.com/notebooks/intro.ipynb?hl=ja "colab") というサービスを通して使用することができます。  
ColaboratoryはGoogleドライブ上のデータを読み込むこともできるので、データをGoogleドライブにアップさえすれば、  
面倒な環境構築作業なしでPythonを通して様々なデータ処理、データ分析が行えます。  
（じゃあColaboratoryで分析すればデータポータルいらないじゃんか、と思われる方もいらっしゃるかもしれませんが、データポータルではウェブ上でグリグリ動かせたりするレポートを簡単に作れるので、Pythonで書くより楽にレポートが作れるという利点があると思います。）  

それではColaboratoryを使ってデータを変形してみましょう。  


### Colaboratoryのセットアップ
Colaboratoryのセットアップは非常に簡単です。  
Googleドライブ上で右クリック→その他→Google Colaboratory を選択すれば使用できるようになります。  
Google Colaboratoryがメニューに表示されない場合は、下の "アプリを追加" から "Colaboratory" を検索してアプリを登録してください。
![colab start](/images/google_dataportal_trans_data/colab_start.png "colab_start")  
画面上部でファイル名が "無題のノートブック.ipynb" となっていると思うので、任意の名前を設定してください。ここでは "melt_data.ipynb" としました。


### ColaboratoryとGoogleドライブを接続する
ColaboratoryでGoogleドライブ上のデータを読み込めるようにします。  
図のように、Colaboratoryの画面左側から "フォルダ" マークをクリックします。
![colab folder](/images/google_dataportal_trans_data/colab_folder.png "colab_folder")  

次に、現れた画面からフォルダのマーク内にGoogleドライブのマークが書かれたところをクリックします。
これでGoogleドライブと接続されます。場合によっては接続に時間がかかることもあります。
![colab mount](/images/google_dataportal_trans_data/colab_mount.png "colab_mount")  

接続されると、 "drive" → "MyDrive" 以下に自分がGoogleドライブに作成したフォルダの一覧が見られるようになります。  
ここからデータの格納先を確認しておきます。
![colab path](/images/google_dataportal_trans_data/colab_path.png "colab_path")  

これでColaboratoryを使用する準備は完了です。


### Colaboratoryでデータ形式の変換スクリプトを書く
それでは、Colaboratoryの中央部分のコードエディタ（1つ1つのブロックをセルと呼びます）にデータを変換するプログラムを書いていきます。  
まずはデータを読み込んで中身を確認してみましょう。   

```Python
import pandas as pd
df = pd.read_csv("./drive/MyDrive/<データを格納しているフォルダ>/MoviesOnStreamingPlatforms_updated.csv")
df.head()
```

1行目でデータ処理用のライブラリ "Pandas" を読み込み、2行目でデータを読み込み、3行目でデータの初め5行を表示しています。  
このコードを実行すると、以下の出力が得られます。 ![original data](/images/google_dataportal_trans_data/original_data.png "original_data")  

次に、データを変形し、その結果を確認します。今回はテストとして全データは使わず、映画タイトルとIMDbの評価点、各視聴サービスごとの視聴可能状況のみを用います。  

```Python
use_cols = ["ID", "Title", "IMDb", "Netflix", "Hulu", "Prime Video", "Disney+"]
id_vars = ["ID", "Title", "IMDb"]
melted_df = pd.melt(df[use_cols], id_vars=id_vars, var_name="Services", value_name="viewable")
melted_df = melted_df.sort_values(["ID", "Services"]).reset_index(drop=True)

melted_df.head(10)
```

1行目では今回使用するデータの列を指定しています。2行目では縦持ちに変換 __しない__ データの列を指定します。  
今回縦持ちに変換するのは各視聴サービスごとの視聴可能状況のみなので、それ以外のデータ列を２行目で指定しています。  
3行目が今回のメインとなる部分です。 `melt` 関数を用いることで列方向に展開されたデータを1つの列にまとめ直しています。  
`melt` 関数についてはこちらで詳しく解説されています。[整然データ(Tidy Data)への変換をpandasでやってみる](https://qiita.com/ishida330/items/922caa7acb73c1540e28 "pandas_melt")  
`var_name` で視聴サービスの名前を格納する列名を `Services` とし、 `value_name` で各視聴サービスごとの視聴可能状況を `viewable` という名前にしています。  
Pandasを使っているのでメインはたった１行だけになってしまいました。  

4行目は必須ではないのですが、IDの数字順と視聴サービスのアルファベット順でデータを並べ替えて多少見やすくしています。  
最後に10行を出力しています。結果は次のようになります。
![long data](/images/google_dataportal_trans_data/long_data.png "long_data")   

さて、これでデータの変形自体はできたので、変形したデータを再度Googleドライブに出力しましょう。

```Python
melted_df.to_csv("./drive/MyDrive/<データを格納しているフォルダ>/movies_melted.csv", index=False
```

今回はデータと同じフォルダに `movies_melt.csv` という名前で保存しました。  


## データポータルで変換後のデータを可視化
それでは、変換後のデータをデータポータルに接続して可視化してみましょう。  
データの接続は [Googleデータポータル入門：第1回]({{<ref "/posts/google_dataportal_setup.md" >}}) を参照してください。  
また、可視化の方法は[Googleデータポータル入門：第2回]({{<ref "/posts/google_dataportal_analysis.md" >}}) を参照してください。  

今回注意すべきなのは、各視聴サービスの視聴可能状況は、 __データ列 `Services` と `viewable` の2つの組み合わせで表現される__ 、という点です。  
そのため、データポータルの可視化の際はディメンションに `Services` と `viewable` の2つを指定します。  
`viewable` を日本語にした指標を作成し、 `IMDb` の上位10%に絞り込むフィルタを作成した結果、以下のようなグラフが作成されました。  
![bar plots](/images/google_dataportal_trans_data/bar_plots.png "bar_plots")  

各視聴サービスごとに同じ可視化の手順を繰り返すことなく、1つのグラフですっきりと目的の集計を表現できました。  


## 最後に
これで今回の内容は終了です。  
Google Analyticsなど、既存のサービスは大抵データを今回説明したようなコンピュータが扱いやすい形式で保持しているので、  
既存のサービスを使う限りはデータ形式についてあまり意識しないで済むかもしれません。  
ただ、自分でデータを色々なところから取得して分析や可視化、レポーティングを行う際は今回説明したようなデータの形式を意識してみると、  
かかる時間が短くなったりミスを減らせるようになったり色々メリットがあると思います。  
皆様も何かしらデータに触れる際はそのデータの形式に少し注目してみてくださいね。  
