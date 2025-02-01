---
layout: post
title: "信号の周波数とDFTの基底の周波数"
date: 2025-02-02
categories:
---

<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.0/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>

信号の周波数がDFTの基底のどの元の周波数とも異なる場合に、パワースペクトルでどの周波数に対応する部分が大きくなるかを調べる。

## 離散フーリエ変換（DFT）

まず、離散フーリエ変換（DFT）について、短時間フーリエ変換（STFT）にも触れつつ簡単に復習する。

複素数の値をとる信号

$$
\begin{align*}
  s\colon \mathbb R & \longrightarrow\mathbb C \\
  t & \longmapsto s(t)
\end{align*}
$$

を考える。ここで、 $$ t \in \mathbb{R} $$ は時刻、 $$s(t)$$ は時刻$$t$$ 秒での信号の値である。

これをサンプル周波数 $$g$$ Hzでサンプリングしたデジタル信号

$$
s_n = s\left(t_n\right) = s\left(\frac{n}{g}\right) \, \left(n \in \mathbb{Z}\right)
$$

を考える。（なお、一般にはサンプリングする時刻は $$t_n = \frac{n}{g} $$ とはならず、位相がずれて $$ t_n = \frac{n + \delta}{g} \, \left(0 \leq \delta < 1\right)$$となる。しかし、どちらの場合でも以下の議論は本質的に変わらないので、表記を簡単にするために本稿では $$ t_n = \frac{n}{g}$$の場合を考える。）

デジタル信号 $$s_n \, \left(n \in \mathbb{Z}\right)$$ を時刻 $$ n = n_0 $$ から長さ $$ N \in \mathbb{N} $$ の窓で切り取った信号 $$ \left(s_{n_0}, \dots, s_{n_0 + N - 1}\right) $$を離散フーリエ変換（DFT）したもの

$$
\sum_{j = 0}^{N - 1} s_{n_0+j} \exp\left(-2\pi i \frac{m}{N}j\right) \, \left(m = 0, \dots, N - 1\right)
$$

を $$ n_0 \in \mathbb{Z} $$ に渡って並べたものを、 $$s_n$$ の短時間フーリエ変換（STFT）という。なお、典型的には信号 $$ \left(s_{n_0}, \dots, s_{n_0 + N - 1}\right) $$ に窓関数を適用してからDFTを適用するが、本稿では窓関数は省略する。

## DFTの基底の周波数
DFTの基底の $$m\,\left(m = 0, \dots N - 1\right)$$ 番目の元

$$
\left( \exp\left( 2\pi i\frac{m}{N}0\right), \dots, \exp\left( 2\pi i\frac{m}{N}n\right), \dots, \exp\left( 2\pi i\frac{m}{N}\left(N-1\right)\right) \right)
$$

は添字 $$n$$ が $$ 0 $$ から $$N$$ まで進む間に $$m$$ 周する。その間に時刻は $$0$$ 秒から $$N/g$$ 秒まで進むので、この元の周波数は $$gm/N$$ Hzである。

たとえば、サンプル周波数が $$g = 16000$$ Hzで、窓の長さが $$N = 2048$$ である場合を考える。このとき、DFTの基底の $$m$$ 番目の元の周波数は $$16000m/2048 = 7.8125m$$ Hzである。

## 信号の周波数とDFTの基底の周波数

以下では、表記の簡単のため、$$n_0 = 0$$ の場合を考える。

また、以下では周波数が $$ f $$ Hzの信号

$$
s(t) = \exp(2\pi ift) \, \left(t \in \mathbb{R}\right)
$$

を考える。サンプル周波数 $$ g$$ Hzでサンプリングしたデジタル信号は

$$
s_n = \exp\left(2\pi i f \frac{n}{g}\right) \, \left(n \in \mathbb{Z} \right)
$$

となる。

いま、信号の周波数を $$f = 440$$Hz、サンプル周波数を $$g = 16000$$Hzとして、このデジタル信号を長さ $$N = 2048$$ の窓で切り取って離散フーリエ変換する場合を考えよう。

このとき、DFTの基底の $$m \, \left(m = 0, \dots, N - 1\right)$$ 番目の元の周波数は前述の通り $$7.8125m$$ Hzである。フーリエ基底の元のなかで周波数が $$440$$ Hzに最も近いものは、 $$m = 56$$ 番目の $$437.5$$Hz の元と、 $$m = 57$$ 番目の $$445.3125$$Hz の元の二つであり、フーリエ基底には信号と同じ $$440$$Hz ぴったりの周波数を持つ元は含まれない。

仮にフーリエ基底の中に信号と全く同じ $$440$$Hz ぴったりの周波数を持つ元が $$m$$ 番目に存在するとすれば、フーリエ基底の直交性によって、DFTした結果は $$m$$ 番目のみが非ゼロでその他の値は全てゼロになることがわかる。一方で、上述の設定では $$440$$Hz ぴったりの周波数を持つ元がフーリエ基底に存在しないので、DFTするとどのような結果になるかはすぐにはわからない。

実際には、 $$440$$Hz に近い $$m=56$$ 番目と $$m=57$$ 番目でパワースペクトルが大きくなることが、STFTを使っていると経験的にはわかる。これを以下で計算して示したい。

## 計算する

再び、信号の周波数 $$f$$Hz 、サンプル周波数 $$g$$Hz、窓の長さ $$N$$ がすべて一般の値の場合を考えよう。ただし、サンプリング定理のため、 $$2f \leq g$$ を満たすとする。

周波数 $$f$$Hz のデジタル信号 $$ \left(\exp\left(2\pi i \frac{f}{g} 0\right), \dots, \exp\left(2\pi i \frac{f}{g} \left(N-1\right)\right)\right) $$を離散フーリエ変換（DFT）したものの $$m$$ 番目の値は、

$$
\sum_{n = 0}^{N - 1} \exp\left(2\pi i \frac{f}{g} n\right) \exp\left(-2\pi i \frac{m}{N}n\right)
$$

と定義される。等比数列の和の公式によって、これは

$$
\frac{\exp\left(2\pi i N \left(\frac{f}{g} - \frac{m}{N}\right)\right) - 1}{\exp\left(2\pi i \left(\frac{f}{g} - \frac{m}{N}\right)\right) - 1}
$$

と計算できる。

なお、この式の分母が $$0$$ になるのは $$\frac{f}{g} - \frac{m}{N}$$ が整数のときであり、 $$0 \leq \frac{f}{g} \leq \frac{1}{2}$$ と $$0 \leq \frac{m}{N} < 1$$ より $$f = \frac{gm}{N}$$ 、すなわち信号の周波数 $$f$$ がフーリエ基底の $$m$$ 番目の元の周波数に一致している場合である。この場合はフーリエ基底の直交性によってDFTの結果は明らかなので、この節では分母が $$0$$ でない場合のみを考える。

$$\exp(2\pi i) = 1$$であるから、この式の分子は

$$
\exp\left(2\pi i N \left(\frac{f}{g} - \frac{m}{N}\right)\right) - 1
= \exp\left(2\pi i \frac{f}{g}N \right) \exp\left(-2\pi im\right) - 1
= \exp\left(2\pi i \frac{f}{g}N \right) - 1
$$

と計算できて、 $$m$$ に依存しない。

$$\left\|\exp\left(i \theta\right) - 1\right\|^2 = 2\left( 1 - \cos \theta \right) \, \left(\theta \in \mathbb{R}\right)$$ より、この式の分母のノルムの2乗は、 $$m$$ が $$0$$ から $$\frac{fN}{g}$$ に向かって増加するにしたがって $$0$$ に向かって減少し、 $$m$$ が $$\frac{fN}{g}$$ から $$\frac{N}{2}$$ に向かって増加するにしたがって増加することがわかる。したがって、デジタル信号の周波数 $$f$$ にフーリエ基底の元の周波数 $$\frac{gm}{N} $$ が近い $$m$$ で、DFTの値のノルムが最大になることがわかる。

## まとめ

以上の計算から、離散フーリエ変換では、フーリエ基底が信号の周波数を含んでいない場合でも、信号の周波数に最も近い周波数成分が最も強く検出されることがわかった。

STFTを使っていて疑問に思っていたが、本をいくつか読んでも疑問が解消せず、自分で計算してみて初めてわかった。自明だったり何かを勘違いしていたりするかもしれないが、いちおう記録・公開しておく。

## 参考文献
- 山下幸彦他『工学のためのフーリエ解析』
- 新井仁之『フーリエ解析とウェーブレット』
