---
layout: post
title: "キーフレーズ抽出アルゴリズムEmbedRankを試した"
date: 2023-11-28
categories: プログラミング
---

キーフレーズ抽出アルゴリズムに興味を持って、「[[1801.04470] Simple Unsupervised Keyphrase Extraction using Sentence Embeddings](https://arxiv.org/abs/1801.04470)」を読んだ。EmbedRankとEmbedRank++というキーフレーズ抽出アルゴリズムが紹介されている。

キーフレーズ抽出は、文や文章からそこに含まれるキーフレーズを抽出する。「キーフレーズ」の定義はよくわからないが、いくつかの文献を眺めてみたところ、「文書の主題を表している語句」という感じで、情報検索や要約といったタスクからの関心がメインでありそうな雰囲気だった。

EmbedRankでは、文章とその候補語句たちを同じ空間に分散表現で埋め込んで、文章全体とコサイン類似度で近いベクトルを持つ候補語句をキーフレーズとして選び出す。候補語句としては、 `(形容詞)*(名詞)+` の形で貪欲にマッチしたものを候補語句としている。

注意点としては、候補語句として `(形容詞)*(名詞)+` のみの形に限っているため、動詞や副詞が含まれた語句はキーフレーズに選ばれることはない。他にも、「freedom of speech」のような前置詞を含む名詞句もキーフレーズにならない。

EmbedRankを実際に実装してみて、[QuicksortのWikipedia記事](https://en.wikipedia.org/wiki/Quicksort)の導入部から上位10個のキーフレーズを求めると、以下のようになった。（ただし、[Inspec](https://huggingface.co/datasets/taln-ls2n/inspec)でベンチマークをとったところ論文のものよりやや低いスコアが出たので実装に間違いがある可能性はある）

1. divide-and-conquer algorithm
2. heapsort
3. algorithm
4. quicksort
5. British computer scientist Tony Hoare
6. sorting
7. in-place
8. comparison sort
9. sort
10. partition-exchange sort

最初の項目が「quicksort」でなく「divide-and-conquer algorithm」になっているのはやや不思議だ。

ところで、この結果を見ると6番目に「sorting」、9番目に「sort」とほとんど同じ意味の語句が選ばれてしまっている。これを解決するのがEmbedRank++で、EmbedRank++では候補語句からキーフレーズを選び出す際にMMR (Maximal Marginal Relevance) を使う。MMRはクエリに近く、かつこれまでに選んだ候補からは遠いものを順次選んでいく方法だ。

MMRを使ったEmbedRank++を、先ほどと同じ文章に適用すると次のようになった。なお、MMRのパラメータλは0.5に設定した。

1. divide-and-conquer algorithm
2. sorting
3. British computer scientist Tony Hoare
4. quicksort
5. larger distributions
6. comparison sort
7. worst case
8. Most implementations
9. type
10. heapsort

今回は「sorting」は出ているが「sort」は選ばれなくなった。全体的に語句の多様性が増している。
