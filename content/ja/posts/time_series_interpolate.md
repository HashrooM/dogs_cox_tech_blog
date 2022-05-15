---
title: "タイムスタンプが欠損した時系列データへの対処"
date: 2022-05-09T21:37:10+09:00
description: "タイムスタンプが欠損している場合の対処方法を説明します"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: DogsCox
tags:
- "時系列"
- "欠損値補間"
series:
- "時系列解析"
categories:
- "データ分析"
---

時系列データにおいて、タイムスタンプが欠損している場合の対処方法です。  
こちらの書籍で書かれている内容を Python でやってみた、という記事です。  
（書籍では R で作業しています。）  

<!-- START MoshimoAffiliateEasyLink -->
<script type="text/javascript">
(function(b,c,f,g,a,d,e){b.MoshimoAffiliateObject=a;
b[a]=b[a]||function(){arguments.currentScript=c.currentScript
||c.scripts[c.scripts.length-2];(b[a].q=b[a].q||[]).push(arguments)};
c.getElementById(a)||(d=c.createElement(f),d.src=g,
d.id=a,e=c.getElementsByTagName("body")[0],e.appendChild(d))})
(window,document,"script","//dn.msmstatic.com/site/cardlink/bundle.js?20220329","msmaflink");
msmaflink({"n":"実践　時系列解析 統計と機械学習による予測 [ Aileen Nielsen ]","b":"","t":"","d":"https:\/\/thumbnail.image.rakuten.co.jp","c_p":"","p":["\/@0_mall\/book\/cabinet\/9601\/9784873119601_1_3.jpg"],"u":{"u":"https:\/\/item.rakuten.co.jp\/book\/16823634\/","t":"rakuten","r_v":""},"v":"2.1","b_l":[{"id":1,"u_tx":"楽天市場で見る","u_bc":"#f76956","u_url":"https:\/\/item.rakuten.co.jp\/book\/16823634\/","a_id":2682976,"p_id":54,"pl_id":27059,"pc_id":54,"s_n":"rakuten","u_so":1}],"eid":"PLg0m","s":"s"});
</script>
<div id="msmaflink-PLg0m">リンク</div>
<br>
<!-- MoshimoAffiliateEasyLink END -->  

時系列データにおいて、タイムスタンプはデータ行のインデックスとして考えられます。  
そのためタイムスタンプの欠損は「あるはずのデータ行の全てが存在しない」という状態に相当します。  

この点が「データ行のうち、一部の列の値が欠損している」という一般的な欠損値と異なる点だと考えられます。  

また、時系列データを取り扱うときは「本来知りえない未来の情報」を分析なりモデルなりに取り込んでしまうことに注意する必要があります。  
（紹介した書籍では "先読み" と言っていたのでこの記事でもそう呼びます。）  

上記の点に留意したうえでタイムスタンプの欠損に対処する方法を説明します。  
コードの実行環境は無料版の Colab です。


## 今回使用するデータ
サンプルデータとして [seaborn の flights データセット](https://github.com/mwaskom/seaborn-data/blob/master/flights.csv "flights") を用います。  
こちらのデータは 1949 年 ～ 1960 年まで、一か月ごとに旅客機の搭乗客数を記録したデータです。  
タイムスタンプは年-月となります。  
以下はライブラリの準備 ~ データの読み込みまでのコードです。

``` Python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
import seaborn as sns
from datetime import datetime
import pprint

sns.set()

df = sns.load_dataset("flights")
```

データは以下のようになっています。

``` Python
> print(df.shape)
> pprint.pprint(df.head(13))

(144, 3)
    year month  passengers
0   1949   Jan         112
1   1949   Feb         118
...
11  1949   Dec         118
12  1950   Jan         115
```

とりあえずデータをプロットしてみます。  
その際に `month` が文字列で与えられていて不便なので、年-月を結合してタイムスタンプ型に直します。

``` Python
# 日付をdatetime型に変換する
df["year-month"] = pd.to_datetime(
    df["year"].astype("str") + "-" + df["month"].astype("str"),
    format="%Y-%b"
)
```

データを出力してみると以下のようになります。  

``` Python
> pprint.pprint(df.head())

   year month  passengers year-month
0  1949   Jan         112 1949-01-01
1  1949   Feb         118 1949-02-01
2  1949   Mar         132 1949-03-01
3  1949   Apr         129 1949-04-01
4  1949   May         121 1949-05-01
```

月初の日付になっていますが、まあ問題ないでしょう。  
ではプロットしてみます。  

``` Python
fix, ax = plt.subplots(figsize=(8, 4))
sns.lineplot(
    data=df, x="year-month", y="passengers", ax=ax
)

# x軸の設定。毎年1月のみプロット
ax.xaxis.set_major_locator(mdates.MonthLocator(bymonth=1))

# x軸の回転
xlabels = ax.get_xticklabels()
plt.setp(xlabels, rotation=45)

plt.tight_layout()
plt.show()
```

![flights](/images/time_series_interpolate/flights.png "flights")  


## タイムスタンプ欠損データの作成
この章は作業の再現性のために書いているので読み飛ばしていただいても大丈夫です。  

`fights` データにはタイムスタンプの欠損は無いので、行をランダムサンプリングすることで無理やりタイムスタンプ欠損の状況を作り出します。  

``` Python
# 未処理のデータフレームを再度読み込む
df = sns.load_dataset("flights")

# 先頭や最後が抜けると元のデータと比較しにくくなるので
# 先頭行と最終行だけ取り出しておく
head1 = df.head(1)
tail1 = df.tail(1)

# 2行目 ~ 最後から2行目までを90%サンプリング
# サンプリングした後に元のindexでソート
df_sampled = df.iloc[1:-1].sample(frac=0.9, random_state=1).sort_index()

# 先頭行と最終行を付け加える
df_sampled = pd.concat([head1, df_sampled, tail1]).reset_index(drop=True)
```

データを出力してみます。  

``` Python
> pprint.pprint(df_sampled)

     year month  passengers
0    1949   Jan         112
..    ...   ...         ...
127  1960   Sep         508
128  1960   Nov         390
129  1960   Dec         432
```

パッと見ただけでも `1960 Oct` が抜け落ちていることが分かります。  
これでタイムスタンプ欠損のあるデータを疑似的に準備できました。  


## タイムスタンプ欠損の確認
実際にどれだけタイムスタンプ欠損があるのか調べてみます。
面倒なのは、タイムスタンプ欠損があるデータであっても列に欠損が無ければ `df.isna()` などではタイムスタンプ欠損を検出できない点です。  
そこでタイムスタンプのラグもしくはリード特徴量（つまりタイムスタンプ列を1行後/先にずらした列）を作成し、その差分を見てみます。  

``` Python
# タイムスタンプの欠損を確認する
# pd.datetime型のカラムを追加
df_sampled["year-month"] = pd.to_datetime(
    df_sampled["year"].astype("str") + "-" + df_sampled["month"].astype("str"),
    format="%Y-%b"
)

# 1行前との月の差分を計算。2以上のものがあると欠損していることがわかる
def calc_month_diff(days1:pd.Series, days2: pd.Series) -> np.array:
  return np.array((days1.dt.year - days2.dt.year) * 12\
                    + (days1.dt.month - days2.dt.month))

timediff_month_lag1 = calc_month_diff(
    df_sampled["year-month"], df_sampled["year-month"].shift(1)
)
```

出力は以下のようになります。

``` Python
# lagとの差分のユニークを出力
> np.unique(timediff_month_lag1)

array([ 1.,  2.,  3., nan])
```

この計算は月の差分をとっていることに相当するので、タイムスタンプ欠損がなければ `1, nan` だけのはずです。  
( `nan` は先頭行にはラグが存在しないために発生します。)  
今は `2, 3` もあるので、一か月飛ばしと二か月飛ばしが発生していることが分かります。  


## 欠損したタイムスタンプの復元
それでは欠損したタイムスタンプを復元してみます。  
方法としてはいくつか考えられますが、今回は以下のようにやってみます。  

1. 欠損のない日付列を作成
2. 作成した日付列をデータフレームのインデックスとして設定

他には欠損のない日付をデータフレームとして作成し、`pd.merge` や `pd.concat` で結合するといった方法も考えられます。  

それでは上記 1, 2 の方法を実装していきます。  


### 1. 欠損のない日付列の作成
まずは欠損のない日付列を作成します。  
日付列は [pd.date_range](https://pandas.pydata.org/docs/reference/api/pandas.date_range.html "date_range") で作成することができます。  

``` Python
# タイムスタンプの欠損を埋めるためのインデックスを生成
all_months_index = pd.date_range(
    start=df_sampled["year-month"].min(),
    end=df_sampled["year-month"].max(),
    freq="MS" # freq引数で日付の間隔を決める
)
```

作成した日付列を出力してみます。  

``` Python
> all_months_index

DatetimeIndex(['1949-01-01', '1949-02-01', '1949-03-01', '1949-04-01',
               ...
               '1960-11-01', '1960-12-01'],
              dtype='datetime64[ns]', length=144, freq='MS')
```

`DatetimeIndex` という型であることが分かります。  
データフレームのインデックスにするのにちょうど良さそうな型名ですね。  


### 2. 作成した日付列をインデックスとして設定
あとはデータフレームの `df.reindex()` メソッドで作成した日付列をインデックスに設定するだけなのですが、  
`df.reindex()` メソッドは元のインデックスと新たなインデックスの間に対応関係が必要なようです。  
どこがどう対応していないといけないかはまだ自分もきちんと理解できていないのですが...  
とりあえず型などがまったく異なると上手くいかないようです。  

今、`df_sampled` データフレームのインデックスは `0, 1, 2, ...` という整数で与えられています。  
一方、新たに設定したいインデックスは日付です。  
このまま `reindex()` を実行しても、インデックス同士の対応が取れず `reindex()` 後のデータフレームの値が全て `NaN` になってしまいます。  
そこで元のデータフレームの方も日付をインデックスにしておきます。  

``` Python
# 元のデータでもdatetimeにしておく必要がある
df_sampled_index_datetime = df_sampled.set_index("year-month")

# date_indexをインデックスに設定した後にreset_indexで普通の整数indexに戻す
# 特に指定していないので欠損値はNaNになる
df_all_months = df_sampled_index_datetime["passengers"].reindex(all_months_index)\
                .reset_index()\
                .rename(columns={"index": "year-month"})
```

タイムスタンプ欠損を復元した結果を出力してみます。  

``` Python
> pprint.pprint(df_all_months)

    year-month  passengers
0   1949-01-01       112.0
..         ...         ...
140 1960-09-01       508.0
141 1960-10-01         NaN
142 1960-11-01       390.0
143 1960-12-01       432.0
```

抜けていた `1960-10-01` の日付がちゃんと復元されていますね。  
このコードでは日付が抜けていた部分の `passengers` の値は `NaN` ですが、 `reindex()` の `method` 引数で `NaN` が生じた場合の補間方法を指定できます。  

- [Pandasでインデックスを再設定するreindex関数の使い方](https://deepage.net/features/pandas-reindex.html "reindex")

実用上はこの `method` 引数を使った方が良さそうですが、ここでは後で別途欠損値を補完する方法を試します。  


## 欠損値の補間
タイムスタンプ欠損を復元したことで `passengers` に欠損値が生じてしまいました。  
ここからはこちらに対処していきます。  

特に時系列データであることを意識せずに通常の欠損値の補間方法（全体の平均で埋めるなど）を使うことも可能ではあります。  
しかし、そうしてしまうと欠損値の補間に「本来使えないはずの未来の値」を利用してることになり、困った状況になり得ます。  

例えば、気温の予測を行うという単純なタスクを考えます。  
1時間置きに気温データを取得し、それまでの値から逐次的に学習、その後の気温の値を予測するというタスクです。  
その際に何らかの異常で最新の気温の値が欠損してしまったとき、欠損の前後の値に依存した補間方法（前後の値で線形補間など）を使っていると、後の値が存在しないのでエラーとなってしまいます。  

また、1時間単位の気温の欠損を埋める時、おそらく1日など長い単位の平均よりも直近の値を用いるほうが補間として適当でしょう。  

このように、時系列データの欠損への対処時は常にデータの先読みに注意する必要があります。  
（もちろん目的によっては先読みしても問題ない場合もあります。）  

以下ではデータを先読みを避けて欠損値を補完する方法について説明します。  


### 移動平均による欠損への対処
直近の何個かの値の平均値で補間する方法です。  
以下は先に処理した `df_all_months` データにおいて、欠損の手前 3 点の値の平均値で欠損を補間するコードです。  
以下の記事を参考にしました。  

- [pandasのresampleとrolling](https://qiita.com/ryskiwt/items/ae1dc7c27374b728f131 "resample_rolling")
- [Pandas の Rolling の使い方](https://qiita.com/propella/items/7667420c2711dc78b433 "rolling")

``` Python
# year-monthをindexにしておく
df_fill_nan_rolling_mean = df_all_months.set_index("year-month")

# 欠損値部分だけ移動平均で埋める関数
def rolling_mean(n):
  is_nan = np.sum(np.isnan(n[-1]))
  if is_nan:
    return np.mean(n)
  else:
    return n[-1]

# 直前3日の移動平均でnanを埋める
df_fill_nan_rolling_mean["passengers_fill_nan_rolling_mean"] =\
  df_fill_nan_rolling_mean["passengers"]\
    .rolling(window=3, min_periods=1)\
    .apply(rolling_mean)\
    .round(3) # 小数点以下3桁までにする
```

補間した結果は以下です。  

``` Python
> pprint.pprint(df_fill_nan_rolling_mean)

            passengers  passengers_fill_nan_rolling_mean
year-month                                              
1949-01-01       112.0                             112.0
...                ...                               ...
1960-09-01       508.0                             508.0
1960-10-01         NaN                             557.0
1960-11-01       390.0                             390.0
1960-12-01       432.0                             432.0
```

想定通り、 `NaN` が直近の移動平均で補間されました。  

ただ、この方法には直近何期分の平均をとるか、という注意点があります。  
今回は元々あったタイムスタンプ欠損がたかだか二か月飛ばしだったので直近 3 期の平均をとりました。  
しかし、より長い欠損が生じうる場合にはこの方法は上手くいきません。  
3 期以上の欠損がある場合、結局そこの値は平均を計算できずに `NaN` のままです。  

また、今回のデータでも二か月飛びの所で上記の方法だと実質欠損直前の値と同じ値で補間されます。  
変化を滑らかにしたい目的で平均を取っているのに、その目的が達成されていないわけです。  

こういった懸念があるため、次で説明する「欠損する直前の値を用いる」という方法を第一候補にするのが良いのではないかと思います。  


### 直前の値を用いた欠損への対処
欠損する直前の値で欠損値を補間する方法です。  
こちらは既に [df.interpolate()](https://note.nkmk.me/python-pandas-interpolate/ "interpolate") メソッドが用意されているのでそれを使うだけです。  

``` Python
# とりあえずコピーを取っておく
df_fill_nan_interpolate = df_all_months.copy()

# 直前の値で補間
df_fill_nan_interpolate["passengers_fill_nan_interpolate"] =\
  df_all_months["passengers"].interpolate("ffill") # "pad"でも同じ
```

補間結果を出力してみます。  


``` Python
> pprint.pprin(df_fill_nan_interpolate)

    year-month  passengers  passengers_fill_nan_interpolate
0   1949-01-01       112.0                            112.0
..         ...         ...                              ...
140 1960-09-01       508.0                            508.0
141 1960-10-01         NaN                            508.0
142 1960-11-01       390.0                            390.0
143 1960-12-01       432.0                            432.0
```

欠損する直前の値で埋められていることが分かります。  
この結果は前述の `reindex()` メソッドの `method` 引数を指定したのと同じです。  

`interpolate()` メソッドでは直前の値で埋める以外にも、

- 線形補間
- スプライン補間
- 直後の値を用いて補間

など色々できます。  
ただ、これらの方法は欠損値の後の値も使うので先読みしても問題ない場合のみ使うのが良いでしょう。