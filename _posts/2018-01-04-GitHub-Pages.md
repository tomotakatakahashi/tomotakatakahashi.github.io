---
layout: post
title: "GitHub Pagesでホームページを作成する"
date: 2018-01-04
categories:
---
GitHub Pagesで自分のウェブページを作成するやり方について簡単に記す。

環境はUbuntu 16.04を仮定する。


## GitHub Pagesを使う

[GitHub Pagesの公式ページにある手順](https://pages.github.com/)にしたがってプロジェクトを作成する。 `index.html` をpushしてうまく動作していることを確認する。


## jekyllをを導入する

みんながよく使っているブログ形式のページをつくるために、[jekyll](https://jekyllrb.com/)を使う。日本語翻訳サイトは情報が古そうなので本家のドキュメントをみるとよい。
Ubuntuに `apt` で `ruby` や `ruby-dev` をインストールしておく。また、 `gem` で `jekyll` と `bundler` を入れる。
[Quick-start guide](https://jekyllrb.com/docs/quickstart/)などを参照するといい。
GitHub Pagesのリポジトリで `jekyll new .` でサンプルを作成し、`bundle exec jekyll serve` でサーバを立ち上げ、 `localhost:4000` にアクセスし、サンプルが見えることなどを確認する。


## jekyllでページを作る

ポストを書きたいときは、[Writing posts\|jekyll](https://jekyllrb.com/docs/posts/)にあるように、 `_posts` 以下にポストを書く。
ウェブサイト全体のタイトルや説明文などは `_config.yml` を設定することで変更できる。



## 参考文献
* [GitHub Pages \| Websites for you and your projects, hosted directly from your GitHub repository. Just edit, push, and your changes are live.](https://pages.github.com/) GitHub Pagesの公式ページ
* [jekyll \| Transform your plain text into static websites and blogs](https://jekyllrb.com/) jekyllの公式ページ
<!--more-->