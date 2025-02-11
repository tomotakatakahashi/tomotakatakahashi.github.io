---
layout: post
title: "Gitのcommitを分割するやり方"
date: 2025-02-22
categories: プログラミング
---

ソフトウェアエンジニアリングにおいて、ソースコードへの変更の差分を小さく分割することは重要です。「[Small CLs \| eng-practices](https://google.github.io/eng-practices/review/developer/small-cls.html)」でも説明されているように、差分を小さくするとコードレビューが効果的になります。また、差分や機能を分割していく過程でプログラムの構造への理解が深まり、よりよいクラス設計に導く効果もあると思います。

ソースコードの管理に[Git](https://git-scm.com/)を使っている場合は、変更はcommitとして記録していくことになります。Gitのコマンド体系は単純ではないので、commitの分割の仕方を習得するのは簡単ではないと思います。そこで、この記事ではcommitを分割するために使えるコマンドを、具体的なユースケースに沿って解説します。

## 1. commitをファイルごとに分ける

<!--
echo A > A.txt
echo B > B.txt
git add A.txt B.txt
git commit -m "Add A and B"
-->

最新のcommitで、次のように2つのファイル `A.txt` と `B.txt` を編集したとします。

```diff
// A.txt
+A
// B.txt
+B
```

このとき、commitをファイルごとに2つに分割してみましょう。

```diff
// A.txt
+A
```

```diff
// B.txt
+B
```

まず、 `git reset --mixed HEAD~` で、working treeはそのままにindexを1つ前のcommitに戻します。次に、 `git add A.txt` で `A.txt` のみをindexに追加します。そして、 `git commit -m "Add A.txt"` で `A.txt` に対する変更のみをcommitします。最後に、 `B.txt` についても同様にcommitします。 `git add B.txt; git commit -m "Add B.txt"` 。

まとめると、以下にようになります。

```shell
git reset --mixed HEAD~
git add A.txt
git commit -m "Add A.txt"
git add B.txt
git commit -m "Add B.txt"
```

これで、commitがファイルごとに分割されました。

## 操作を間違えたときに元の状態に戻す

途中で操作を間違えて、変更を失ってしまったときの復元方法についても説明しておきます。

`git reset --mixed HEAD~` したあとに、間違えて `rm A.txt` を実行してしまい、 `A.txt` への変更を消してしまったとしましょう。

そんなときは、 `git reflog` を実行すると最近の操作履歴を見ることができます。一番上の2行がだいたい以下のような内容になっているので、復元したいcommitのハッシュを覚えておきます。この場合は `f1a1c1a` です。

```
294bd38 (HEAD -> master) HEAD@{0}: reset: moving to HEAD~
f1a1c1a HEAD@{1}: commit: Add A and B
```

そして、 `git reset --hard f1a1c1a` コマンドでindexとworking treeの両方を復元したいcommitの状態に戻します。あるいは、今回の場合は戻りたいcommitは `HEAD` の1つ前の状態 `HEAD@{1}` なので、commitのハッシュ `f1a1c1a` の代わりに `HEAD@{1}` を使うこともできます。 `git reset --hard HEAD@{1}` 。

まとめると、以下のようになります。

```shell
git reflog # 履歴から所望のcommit hashを調べる
git reset --hard ${commit_hash}
```

## 2. ファイルの部分ごとにcommitを分ける

最新のcommitで、次のようにファイル `A.txt` を編集したとします。

```diff
// A.txt
 A1
+A2
+A3
```

このとき、commitを行ごとに2つに分割してみましょう。

```diff
// A.txt
 A1
+A2
```
```diff
// A.txt
 A1
 A2
+A3
```

まず、 `git reset --mixed HEAD~` で、indexのみを1つ前のcommitに戻します。次に、 `git add --patch A.txt` を実行します。すると、以下のような対話モードに入ります。

```
@@ -1 +1,3 @@
 A1
+A2
+A3
(1/1) Stage this hunk [y,n,q,a,d,e,p,?]?
```

ここで `e` と入力するとテキストエディタが開かれるので、以下の「+A3」の行を削除し、ファイルを保存して、テキストエディタを終了します。

```
@@ -1 +1,3 @@
 A1
+A2
+A3
```

すると、「A2」の行のみがindexに入った状態になるので、commitします。 `git commit -m "Add A2"` 。次に、 `git add .; git commit -m "Add A3"` として、「A3」の行もcommitします。

まとめると、以下のようになります。


```shell
git reset --mixed HEAD~
git add --patch A.txt
# 「A2」の行のみをindexに追加する
git commit -m "Add A2"
git add .
git commit -m "Add A3
```

これで、commitが行ごとに分割されました。

ファイルの一部分だけをindexに追加するためのコマンド `git add --patch` の部分の操作が少し面倒で難しいです。これは、素の `git` コマンドでなく、気の利いたGitクライアントを使うとより簡単にできるはずです。筆者はEmacs用のGitクライアント[Magit](https://magit.vc/)を使っており、Magitなら行やhunkごとの `git add` がもっと直感的な操作でできます。Magit以外のGitクライアントでも、きっと直感的な操作方法が提供されていると思います。

## 3. HEADより前のcommitを分ける

次は、最新でないcommitを分割する方法を解説します。1つ目のcommitで `A.txt` と `B.txt` を編集して、2つ目のcommitで `C.txt` を編集したとします。したがって、Gitの `HEAD` は `C.txt` を編集したcommitを指しているとします。

```diff
// A.txt
+ A
// B.txt
+ B
```
```diff
// C.txt
+ C
```

<!--
echo A > A.txt
echo B > B.txt
git add A.txt B.txt
git commit -m "Add A.txt and B.txt"
echo C > C.txt
git add C.txt
git commit -m "Add C.txt"
-->

この2つのcommitを、 `A.txt`, `B.txt`, `C.txt` の3つのcommitに分割したいとします。

```diff
// A.txt
+ A
```
```diff
// B.txt
+ B
```
```diff
// C.txt
+ C
```

いま、 `HEAD` は `C.txt` を追加したcommitを指しているので、「1. commitをファイルごとに分ける」で説明したように `git reset --mixed HEAD~` をしても `C.txt` のcommitが巻き戻されるだけで、分割したい `A.txt` と `B.txt` のcommitは巻き戻されません。

そんなときは、 `git rebase --interactive` を使います。 `A.txt` と `B.txt` を追加したcommitの直前のcommit、すなわち `HEAD` の2つ前のcommitからrebaseするため、 `git rebase --interactive HEAD~2` を実行します。すると、テキストエディタが起動して、以下のようなファイルが表示されます。

```
pick ae7b22f Add A.txt and B.txt
pick 35c9ab6 Add C.txt
```

これを、以下のように1つ目のcommitを「pick」から「edit」に変更し保存して、テキストエディタを終了します。

```
edit ae7b22f Add A.txt and B.txt
pick 35c9ab6 Add C.txt
```

すると、 `A.txt` と `B.txt` のcommitの直後で修正のために止まります。そこで、「1. commitをファイルごとに分ける」と同様に最新のcommitを分割します。すなわち、 `git reset --mixed HEAD~; git add A.txt; git commit -m "Add A.txt"; git add B.txt; git commit -m "Add B.txt"` と実行していきます。

目的のcommitの修正が済んだので、 `git rebase --continue` をして、 `C.txt` のcommitを反映します。これで、目的のcommitの分割ができます。

まとめると、以下のようになります。


```shell
git rebase --interactive HEAD~2
# 1つ目のcommitを「pick」から「edit」に変更する
git reset --mixed HEAD~
git add A.txt
git commit -m "Add A.txt"
git add B.txt
git commit -m "Add B.txt"
git rebase --continue
```

## 4. commitを分割するために修正が必要なとき

最後に、commitを分割するときに、diffの行を振り分けるだけでなく、一部手作業での修正が必要な場合の操作方法を説明します。

いま、「A1」と「A2」を追加する1つのcommitがあったとします。

```diff
// A.txt
+A1
+A2
```

このとき、このcommitを、「A」を追加するcommitと、「A」を消して「A1」と「A2」を追加するcommitに分割したい場合を考えましょう。

```diff
// A.txt
+A
```

```diff
// A.txt
-A
+A1
+A2
```

これは「2. ファイルの部分ごとにcommitを分ける」の場合と似ていますが、単にcommitのdiffを行ごとに分割するのではなく、「A」だけという中継状態を一度挟む必要があるところが違います。

まずは、これまで説明した方法で、「A1」と「A2」の2つのcommitに分割します。これはできたとしましょう。

```diff
// A.txt
+A1
```

```diff
// A.txt
 A1
+A2
```

<!--
echo A1 > A.txt
git add A.txt
git commit -m "Add A1"
echo A2 >> A.txt
git add A.txt
git commit -m "Add A2"
-->

「A1」のcommitを修正して「A」のcommitを作るために、 `git rebase --interactive` を使います。rebaseの始点は2つ前のcommit `HEAD~2` にして、 `git rebase --interactive HEAD~2` を実行します。テキストエディタが起動して以下のようなファイルが開かれます。

```
pick 58db30e Add A1
pick c6eb6ad Add A2
```

これを以下のように編集・保存してテキストエディタを終了すると、「Add A1」の直後で修正のために停止します。

```
edit 58db30e Add A1
pick c6eb6ad Add A2
```

いま、 `A.txt` のファイルの中身は「A1」ですが、1つ目のcommitとしてほしい差分は「A」なので、 `A.txt` を編集して上書きしましょう。 `echo A > A.txt`。これをcommitします。 `git add A.txt; git commit -m "Replace A1 with A"` 。

一方で、最終的な望ましい状態は「A」でなく「A1」なので、 `git revert HEAD` を実行して「Replace A1 with A」の変更を巻き戻します。

そして、 `git rebase --continue` で、「A2」のcommitを反映します。

ここまでの作業で、4つのcommitが生成されています。

```diff
// Add A1
+A1
```

```diff
// Replace A1 with A
-A1
+A
```

```diff
// Revert "Replace A1 with A"
-A
+A1
```

```diff
// Add A2
 A1
+A2
```

最後に、もう一度 `git rebase --interactive` を使ってcommitを整理しなおします。rebaseの起点として「Add A1」の直前のcommitを指定するため、 `git rebase --interactive HEAD~4` を実行します。テキストエディタが起動するので、以下のように2つ目と4つ目のcommitを「pick」から「squash」に書き換えます。

```
pick 58db30e Add A1
squash d0c2009 Replace A1 with A
pick f7c71d5 Revert "Replace A1 with A"
squash b3ac1f5 Add A2
```

保存してテキストエディタを終了すると、1つ目と2つ目のcommitを合体したcommitのmessageを編集するためのテキストエディタが起動します。テキストエディタには

```
# This is a combination of 2 commits.
# This is the 1st commit message:

Add A1

# This is the commit message #2:

Replace A1 with A
```

というテキストが表示されるので、これを

```
Add A
```

と書き換えて、保存・終了します。これで、1つ目と2つ目のcommitを合体したcommit「Add A」ができます。

続いて、3つ目のcommitと4つ目のcommitを合体したcommitのmessageを編集するためのテキストエディタが起動します。エディタに

```
# This is a combination of 2 commits.
# This is the 1st commit message:

Revert "Replace A1 with A"

This reverts commit d0c2009e11a0842536edd04ce2cd8a7faf89cdaa.

# This is the commit message #2:

Add A2
```

と表示されているはずなので、これを

```
Replace A with A1 and A2
```

と書き換えて、保存・終了します。

すると、以下の2つのcommitに分割されます。これで作業は完了です。

```diff
// Add A
+A
```

```diff
// Replace A with A1 and A2
-A
+A1
+A2
```

まとめると、以下のようになります。

```shell
# あらかじめ「A1」のcommitと「A2」のcommitに分割しておく
git rebase --interactive HEAD~2
# 「A1」のcommitをeditにする
echo A > A.txt
git add A.txt
git commit -m "Replace A1 with A"
git revert HEAD
git rebase --continue
git rebase --interactive HEAD~4
# 2つ目と4つ目のcommitを「pick」から「squash」にする
```

## テストを走らせたいとき

分割をしたそれぞれのcommitでテスト・lint・formatなどが通ることを確かめるには、2つの方法があります。

1つ目はcommitの分割作業中に `git stash` を使ってテストを変更を退避し、テストを走らせる方法です。たとえば、「1. commitをファイルごとに分ける」で「Add A.txt」のcommitの直後にテストを走らせるには、以下のようにします。 `git stash push` でcommitしていない変更を退避させ、テストコマンドを実行し、退避させた変更を `git stash pop` で戻すということです。

```shell
git reset --mixed HEAD~
git add A.txt
git commit -m "Add A.txt"
git stash push --include-untracked
pytest # or whatever you want to run
git stash pop
git add B.txt
git commit -m "Add B.txt"
```

2つ目は、commitの分割が終わってからそれぞれのcommitに対してテストを走らせる方法です。 `git rebase --interactive` でcommitを「pick」から「edit」にするとそのcommitの直後で止まるので、すべてのcommitを「edit」にして、各commitに対してテストを走らせ `git rebase --continue` を繰り返すことができます。「1. commitをファイルごとに分ける」を例にあげると、以下のようになります。

```shell
git reset --mixed HEAD~
git add A.txt
git commit -m "Add A.txt"
git add B.txt
git commit -m "Add B.txt"

git rebase --interactive HEAD~2
# 2つのcommitを両方とも「pick」から「edit」にする
pytest # or whatever you want to run
git rebase --continue
pytest # or whatever you want to run
git rebase --continue
```

## 終わりに
この記事では、Gitのcommitを分割するやり方を解説しました。commitを分割するにあたって、以下のようなコマンドが役立つことを説明しました。

- `git reset --mixed`
- `git reflog`
- `git reset --hard`
- `git add --patch`
- `git rebase --interactive`
- `git revert`
- `git stash`

Pull Requestやcommitを分割するように依頼する機会が仕事中などにありますが、参考資料として渡しやすい記事が見つけられなかったので、筆者の知る範囲でやり方を書きました。より具体的には、単に「rebaseしてeditすればよい」というだけでなく、いろいろなユースケースを網羅するようにしました。

## 参考
- [Git - 歴史の書き換え](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E6%AD%B4%E5%8F%B2%E3%81%AE%E6%9B%B8%E3%81%8D%E6%8F%9B%E3%81%88)
- [Small CLs \| eng-practices](https://google.github.io/eng-practices/review/developer/small-cls.html)
- [Write Better Commits, Build Better Projects - The GitHub Blog](https://github.blog/developer-skills/github/write-better-commits-build-better-projects/)
