---
layout: post
title: "常微分方程式を数値的にオイラー法で解く"
date: 2018-11-04
categories: 数値解析

---
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.0/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>


　常微分方程式を計算機で解く方法を解説する。
これは齊藤宣一『数値解析』の第4章の読書ノートである。


## 4.1  初期値問題とオイラー法
　常微分方程式の初期値問題

$$ \begin{array}{ccc} u(0) &=& a \\ \frac{du}{dt}(t) &=& f(t, u(t)) \end{array} $$

を考える。ただし、 関数 $$ f(t, y) $$ は $$ y $$ に関してリプシッツ連続であるとする。すなわち、正数 $$\lambda > 0$$ が存在して、
任意の $$t, y, z$$ に対して $$\left| f(t, y) - f(t, z) \right| \leq \lambda\left| y-z \right| $$ が成り立つとする。
また、この微分方程式が閉区間上でなめらかな解 $$ u\colon [0, T] \to \mathbb{R}$$ をもつとする。

　この常微分方程式の数値解法で最も基礎的なものは、オイラー法と呼ばれるものである。オイラー法は、解関数 $$u(t)$$ を離散化し、常微分方程式に対応する差分方程式を解く。

### オイラー法
　自然数 $$N \in \mathbb{N}$$ をとり、 $$h := T/N, t_n := nh$$ とする。数列 $$ \{U_n\}_{n=0}^N $$ を漸化式

$$ \begin{array}{ccc} U_0 &=& a \\ \frac{U_{n+1} - U_n}{h} &=& f(t_n, U_n) \end{array} $$

によって定める。これを解くと $$ U_n \approx u(t_n) $$ となることが期待される。この解法をオイラー法という。


### 定理（オイラー法の誤差評価）
　上記の設定のもとで、ある正の定数 $$B_2$$ が存在して、任意の $$n (0 \leq n \leq N) $$ に対して、数値解の誤差 $$E_n := u(t_n) - U_n$$ は

$$ |E_n| \leq B_2 \frac{e^{T\lambda} - 1}{\lambda} h$$

を満たす。

#### 証明
　平均値の定理より、 $$\xi \in [t_n, t_{n+1}]$$ が存在して、

$$ \begin{eqnarray} E_{n+1} - E_n &=& [u(t_{n+1}) - u(t_n)] - [U_{n+1} - U_n] \\ &=& [u'(t_n)h + \frac{1}{2}u''(\xi)h^2] - hf(t_n, U_n) \\ &=& h[f(t_n, u(t_n)) - f(t_n, U_n)] + \frac{1}{2} u''(\xi)h^2\end{eqnarray} $$

が成り立つ。
　リプシッツ連続性より、

$$ \begin{eqnarray}|E_n| &=& \left|E_{n-1} + h [f(t_{n-1}, u(t_{n-1})) - f(t_{n-1}, U_{n-1})] + \frac{1}{2} u''(\xi)h^2 \right| \\ &\leq & |E_{n-1}| + h\lambda|u(t_{n-1}) - U_{n-1}| + \frac{1}{2}\|u''\|_{\max} h^2\\ &=& \left| E_{n-1} \right| (1+h\lambda) + \frac{1}{2}\|u''\|_\max h^2\end{eqnarray}$$

が成り立つ。ここで、 $$ B_2 := \frac{1}{2} \| u''\|_{\max} $$ とおく。すると、

$$ \begin{eqnarray} \left| E_n \right| &\leq& \left| E_{n-1} \right| (1+h\lambda) + B_2 h^2 \\ &\leq& \cdots \\ &\leq& |E_0|(1+h\lambda)^n + B_2 h^2\frac{(1+h\lambda)^n -1}{(1+h\lambda) - 1} \\&=& B_2 \frac{(1+h\lambda)^n - 1}{\lambda}h \\&\leq& B_2 \frac{e^{nh\lambda}-1}{\lambda}h \\&\leq&B_2\frac{e^{T\lambda}-1}{\lambda}h\end{eqnarray} $$

と計算できる。

## 参考文献

* 齊藤宣一『数値解析』
