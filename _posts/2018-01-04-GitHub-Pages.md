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

### 2023-11-04 追記
[Jekyll on macOS | Jekyll • Simple, blog-aware, static sites](https://jekyllrb.com/docs/installation/macos/) にしたがってJekyllをインストールしようとすると、 `ruby-install ruby 3.1.3` をした段階で

> ossl_ts.c:829:5: error: incomplete definition of type 'struct TS_verify_ctx'

というエラーが出た。これは環境変数を

```
export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib"
export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include"
export PKG_CONFIG_PATH="/opt/homebrew/opt/openssl@1.1/lib/pkgconfig"
```

と設定すれば回避できた。（事前に `brew install openssl@1.1` する必要もあるかもしれない。）参考リンク：[Compiling ruby 3.1.3 failed! (MacOS Ventura 13.5.1 M1 Max) · rbenv/ruby-build · Discussion #2245](https://github.com/rbenv/ruby-build/discussions/2245)

## jekyllでページを作る

ポストを書きたいときは、[Writing posts\|jekyll](https://jekyllrb.com/docs/posts/)にあるように、 `_posts` 以下にポストを書く。
ウェブサイト全体のタイトルや説明文などは `_config.yml` を設定することで変更できる。



## 参考文献
* [GitHub Pages \| Websites for you and your projects, hosted directly from your GitHub repository. Just edit, push, and your changes are live.](https://pages.github.com/) GitHub Pagesの公式ページ
* [jekyll \| Transform your plain text into static websites and blogs](https://jekyllrb.com/) jekyllの公式ページ
<!--more-->
