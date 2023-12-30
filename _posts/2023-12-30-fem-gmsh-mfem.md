---
layout: post
title: "GmshとMFEMで有限要素法を試す"
date: 2023-12-30
categories: プログラミング
---

<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.0/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>

有限要素法に興味を持って、入門的な勉強をしていた。有限要素法を実行するためのOSSはいくつかあるようだが、使い方が難しかったので、ここに使い方を書いておく。ここでは[Gmsh](https://gmsh.info/)と[MFEM](https://mfem.org/)を使う。

私の環境は、MacBook Pro 14-inch, 2021、macOS Venturaだ。


## Gmshで図形を定義する

Gmshは、[ウェブサイト](https://gmsh.info/)の説明によれば

> Gmsh is an open source 3D finite element mesh generator with a built-in CAD engine and post-processor.

ということで、要するに有限要素法で使うメッシュを生成してくれるソフトウェアのようだ（ただし、あとで使うMFEMにもメッシュを操作する機能はありそうな雰囲気だが、私は違いをよくわかっていない）。

Gmshのインストール手段としては、[Homebrewのformula](https://formulae.brew.sh/formula/gmsh)も用意されているようだが、ここではPythonのインターフェイスを使うので、pipでインストールすればいいと思う。私の環境ではバージョン4.11.1がインストールされている。

```bash
pip install gmsh
```

図形やメッシュの情報を持ったファイルを、Gmshで生成する。このファイルはあとでMFEMの入力になる。ここでは単純な円盤を作る。

```python
import gmsh
import sys

gmsh.initialize()

gmsh.model.add("circle")

lc = 0.05
gmsh.model.geo.add_point(+1., 0, 0, lc, 1)
gmsh.model.geo.add_point(0, +1., 0, lc, 2)
gmsh.model.geo.add_point(-1., 0, 0, lc, 3)
gmsh.model.geo.add_point(0, -1., 0, lc, 4)
gmsh.model.geo.add_point(0, 0, 0, lc, 5)

gmsh.model.geo.add_circle_arc(1, 5, 2, 1)
gmsh.model.geo.add_circle_arc(2, 5, 3, 2)
gmsh.model.geo.add_circle_arc(3, 5, 4, 3)
gmsh.model.geo.add_circle_arc(4, 5, 1, 4)

gmsh.model.geo.addCurveLoop([1, 2, 3, 4], 1)

gmsh.model.geo.addPlaneSurface([1], 1)

gmsh.model.geo.synchronize()

gmsh.model.addPhysicalGroup(2, [1], name = "My surface")

gmsh.model.mesh.generate(2)

gmsh.write("circle.msh2")

if '-nopopup' not in sys.argv:
    gmsh.fltk.run()

gmsh.finalize()
```

このスクリプトを実行すると、 `circle.msh2` ファイルが保存される。GmshのGUIウィンドウも表示される。

![GmshのGUI]({{site.baseurl}}/images/2023-12-30-fem-gmsh-mfem/gmsh.png)

このプログラムは、[公式ドキュメント](https://gmsh.info/doc/texinfo/gmsh.html)の `t1.py` （[GitHub](https://gitlab.onelab.info/gmsh/gmsh/-/blob/4608e03c213fb6d19fb9561dfe47d7261c20e872/tutorials/python/t1.py)）に基づいて書いた。不明な点があれば、ドキュメントを参照するといい。

注意するべき点が、拡張子が `.msh` でなく `.msh2` になっていることだ。拡張子を `.msh` にしてしまうとあとでMFEMで読み込めないファイルが生成されてしまうようなので、このようにしている。

## MFEMで有限要素法を実行する

MFEMは[ウェブサイト](https://mfem.org/)の説明によれば

> MFEM is a free, lightweight, scalable C++ library for finite element methods.

ということで、有限要素法のライブラリだ。

MFEMの[入手方法](https://mfem.org/building/)はいろいろあり、[Homebrewのformula](https://formulae.brew.sh/formula/mfem)も用意されているようだが、ここでは**ソースコードをダウンロードしてビルドをする**方法を使う必要がある。なぜなら、ソースコードに同梱されている[Example Codes](https://mfem.org/examples/)を利用したいが、Homebrew版だとExample Codesがおそらく使いづらいからだ。Serial versionとParallel versionをどちらもexamplesまでビルドしておく。

私の環境では、 `glvis-4.2` 、`mfem-4.6` 、`metis-4.0.3` 、 `hypre-2.26.0` が入っている。

可視化のため、引数無しでGLVisを起動しておく。19916番ポートで可視化のための入力を待ち受けるようになる。

```
cd glvis-4.2
./glvis
```

次に、MFEMの[Example 11](https://mfem.org/examples/#example-11-laplace-eigenproblem)を実行する。

```bash
cd mfem-4.6/examples
./ex11p -m your/path/to/circle.msh2
```

すると、MFEMが有限要素法を実行し、GLVisがMFEMの解析結果を可視化してくれる。

![第一固有関数のGLVisによる可視化]({{site.baseurl}}/images/2023-12-30-fem-gmsh-mfem/mfem1.png)

MFEMを起動しているターミナルに戻りcキーを押すと、次の固有値に対応する固有関数が可視化される。マウスでドラッグすることでいろいろな角度から関数を見ることもできる。

![第二固有関数のGLVisによる可視化]({{site.baseurl}}/images/2023-12-30-fem-gmsh-mfem/mfem2.png)

MFEMのExample 11は方程式 $$-\Delta u = \lambda u$$ をディリクレ境界条件で解くプログラムのようだ。細かい説明は省略するが、これは、円盤型の太鼓（＝普通の太鼓）の膜の振動を調べていると解釈できる。

## まとめ

GmshとMFEMを使って有限要素法を実行する方法について、簡単に説明した。
