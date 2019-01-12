---
layout: post
title: "JavaScriptでゲームを作った"
date: 2019-01-12
categories: プログラミング
---

 JavaScriptでゲームを作った。
<div id="score">score: 0</div>
<div id="game-aria"></div>

 「手でさわれるテトリス」というテーマでつくった。まだ作り込みが足りない部分もあって、バグが残っていたり、グラフィックやエフェクトがなかったり、得点のSNS共有ボタンがなかったり、ソースコードが汚かったり、というのがあるが、時間と気力の問題でここまでにしておく。

[ゲーム単体のページ](https://tomotakatakahashi.github.io/handtris/)もある。

## 技術的な話
 JavaScriptと[Matter.js](http://brm.io/matter-js/)を使ってつくった。JavaScriptを選んだのは、ブラウザで動くようにしておけばPCからもスマートフォンからも遊べるため。Matter.jsはWebのための2次元物理エンジンで、デモが豊富に用意されているので使い方がわかりやすい。
 
 JavaScriptはWeb開発をするならば必ず触ることになるので、この機会に練習したかった。

当初はテトリミノがぐにゃぐにゃ動くように作ろうと思ったのではなくて、テトリミノ一つ一つが剛体としてふるまうようなゲームを作ろうとしていた。剛体であるテトリミノをクリックすると、剛体上の1点をつかむことができて、そのままドラッグすると振り回して回転させられるようにしたかった。しかし、Matter.jsで剛体をつくると、ドラッグしたときに剛体の回転には影響を与えずに剛体全体が移動するようにしかできないみたいだったので、テトリミノを正方形4つを鎖でつないだものとして表現したところ、このようにぐにゃぐにゃになった。よく考えてみると、剛体の1点をつかんで移動させた状態をシミュレーションするのは大変な気もする。

描画にもう少し凝りたかったが、Canvasがよくわからなかったし、調べる気力も残っていなかった。

JavaScriptでゲームをつくると、GitHubとGitHub Pagesで全てホスティングできるので、楽だということがわかった。ただし、得点のランキング機能など、サーバサイドのプログラムを走らせたい場合は別かもしれない。

## 感想
 数名の知人に見せたら、おもしろおかしいという反応をもらうことができたので、満足した。

JavaScriptはIEだと[`for...of`文](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for...of)にも対応していなかったりして、仕事でブラウザ互換などを気にしながらJavaScriptを書く人は大変そうだとおもった。

<script src="https://tomotakatakahashi.github.io/handtris/matter.js"></script>
<script src="https://tomotakatakahashi.github.io/handtris/myscript.js"></script>
