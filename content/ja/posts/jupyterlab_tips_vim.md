---
title: "JupyterLabのターミナルから開くvimはどこのvimか"
date: 2021-05-22T22:14:58+09:00
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: DogsCox
tags:
- JupyterLab
- Vim
series:
- 
categories:
- Tips
---

あまり使い所がないTipsのような気がしますが、ちょっと発見したので備忘録として残しておきます。  

JupyterLabは（言わずもがなですが）ウェブ上で操作、編集できます。  
そして、JupyterLabはシステムのターミナルを開くことができます。  
この２つを合わせると、ウェブ上でVimを使うことができるのでは？と思いつきました。  
とはいえブラウザ上で動作するという特性上、システムのvimをそのまま動かしているのだろうか...と気になります。  

そこで今回以下のような簡単な実験を行ってみました。  

1. Vimにvim-plugin (今回はNerdTree)をdeinでインストール
    - deinについては [こちら](https://github.com/Shougo/dein.vim "dein") を参照
2. JupyterLabを開いて、ターミナルからVimを起動
    - JupyterLabについては [こちら](https://qiita.com/kirikei/items/a1639954ce5ccaf7ac3c "jupyterlab") を参照
3. 果たしてJupyterLabのターミナル上で開いたVimからNerdTreeを実行することはできるのだろうか？

NerdTreeにしたのはパッと見で明らかにvim-pluginが入ったかどうかがわかるので採用しました。JupyterLabにはファイルエクスプローラーがはじめからついているので、実用性は皆無です。  

結果、普通にNerdTreeを開けました。  
![jupyterlab nerdtree](/images/jupyterlab_tips_vim/jupyterlab_vim_nerdtree.png "jpyter_nerdtree")  

これを使えばサーバ上でJupyterLabを起動し、Vimで色々プラグインを導入することでウェブ上で動作する最強の開発環境を作れる...かも？  
