---
layout: post
title: "数値積分入門"
date: 2018-05-19
categories: 数値解析

---
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.0/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>


関数の定積分を計算機で計算する方法を解説する。
齊藤宣一『数値解析』の第2章の読書ノートである。


## 2.1 複合中点公式
定積分を計算するには、リーマン和を計算する方法が基本である。リーマン和を計算する際、代表点として区間の中点を採用する次の公式は中点公式と呼ばれる。

### 定義（中点公式）
$$f: [a, b] \to \mathbb{R}$$を連続関数とする。

$$M(f) := f(\frac{a+b}{2})(b-a)$$を中点公式、

$$M_h(f) := \sum_{j=1}^n f(x_{j-\frac{1}{2}})h$$を複合中点公式という。

ただし、$$n\in\mathbb{N}$$を自然数、$$h=\frac{b-a}{n}$$を区間の幅、$$x_j := a + jh$$をとおく。

中点公式の誤差評価に関しては、以下が成り立つ。

### 定理（中点公式の誤差）
$$f :[a, b]\to \mathbb{R}$$が$$C^2$$級であるとする。このとき、以下が成り立つ。

$$\left | \int_a^b f(x)dx - M(f) \right| \leq \frac{1}{24}\Vert  f’’ \Vert_\infty (b-a)^3$$

$$\left | \int_a^b f(x)dx - M_h(f) \right| \leq \frac{1}{24}\Vert f’’\Vert_\infty (b-a) h^2$$

ただし、$$\Vert f’’ \Vert _\infty := \sup\{f’’(x) \mid x \in [a, b]\}$$である。


#### 証明

テイラーの定理より、

$$f(x) = f(c) + f’(c)(x-c) + \frac{1}{2}f’’(\xi)(x-c)^2$$

とかける。ただし、$$c := \frac{a+b}{2}$$で、$$\xi$$は$$x$$と$$c$$の間の数である。

したがって、

$$\begin{align*}&\int_a^b f(x)dx - f(c)(b-a)\\ =& \left[ f(c)(b-a) + f’(c)\int_a^b (x-c)dx +\frac{1}{2}f’’(\xi)(x-c)^2dx\right] -f(c)(b-a)\\ =& \frac{1}{2}\int_a^bf’’(\xi)(x-c)^2dx\end{align*}$$

となるので、

$$\begin{align*}&\left\vert\int_a^bf(x)dx - M(f)\right\vert \\ \leq & \frac{1}{2}\Vert f’’ \Vert_\infty \int_a^b (x-c)^2dx \\=& \frac{1}{24}\Vert f’’\Vert_\infty (b-a)^3\end{align*}$$

がわかる。定理の2番目の式については、この式の$$a, b$$を$$x_j, x_{j+1}$$として考えて足し合わせることで、

$$\begin{align*} &\left\vert \int_a^b f(x)dx - M_h(f)\right\vert \\ \leq & n \times \frac{1}{24}\Vert f’’ \Vert_\infty h^3 \\ = & \frac{1}{24}\Vert f’’\Vert_\infty (b-a)h^2\end{align*}$$

をえる。（証明終わり）

### 注意
$$ R_h^l (f) := \sum_{j=1}^n f(x_{j-1})h$$と定義すると、誤差評価は

$$\left\vert \int_a^b f(x)dx - R_h^l(f)\right\vert \leq \frac{1}{2} \Vert f’ \Vert_\infty (b-a)h$$

となり、$$M_h(f)$$と比べて収束が悪いことがわかる。

### 注意
1次関数の数値積分を考えると、中点則では誤差が0になるが、$$R_h^l(f)$$では、1次関数が誤差評価の最悪の場合の例を与える。


## 2.2 複合台形則と複合シンプソン則

中点則ではリーマン和と同じようにグラフを長方形型に分割したが、台形型に分割して積分を計算する複合台形則というものもある。

### 定義（台形則）

$$ T(f) := \frac{f(a)+f(b)}{2} (b-a) \mathrm{（台形則）}$$

$$T_h(f) := \sum_{j=1}^n \frac{f(x_{j-1}) + f(x_j)}{2} h \mathrm{（複合台形則）} $$

#### 注意
台形則の誤差評価は中点則の2倍悪い。

##### 事実

$$\left\vert \int_a^bf(x)dx - T(f)\right\vert \leq \frac{1}{12}\Vert f’’\Vert_\infty (b-a)^3$$

中点法則と台形則ともに、2次関数の$$\mathrm{頂点}\pm h/2$$上での積分が、誤差評価の最悪の場合の例を与える。


中点則は関数の0次近似の積分、台形則は関数の1次近似の積分と考えることができる。シンプソン則は、関数の2次近似の積分である。これは、端点と中点をあわせた3点に対してラグランジュ補間を考えると理解することができる（詳細は省略）。

### 定義（シンプソン則）

$$S(f):=\frac{f(a)+4f(c)+f(b)}{6}(b-a)\, \mathrm{（シンプソン則）}$$

$$S_h(f) := \sum_{j=1}^n\frac{f(x_{j-1})+4f(x_{j-\frac{1}{2}}) +f(x_j)}{6}h \, \mathrm{（複合シンプソン則）}$$

複合シンプソン則の誤差評価は中点則よりもよい。

### 定理（シンプソン則の誤差）

$$f : [a, b] \to \mathbb{R}$$が$$C^4$$級であったとする。このとき、以下が成り立つ。

$$\left\vert \int_a^bf(x)dx - S(f)\right\vert \leq \frac{1}{2880}\Vert f^{(4)}\Vert_\infty(b-a)^5$$

$$\left\vert \int_a^bf(x)dx - S_h(f)\right\vert \leq \frac{1}{2880}\Vert f^{(4)}\Vert_\infty(b-a)h^4$$

#### 証明
簡単のため、$$x$$軸方向に平行移動して、
$$f: [-k, k] \to \mathbb{R}$$
としても一般性を失わない。ここで、$$2k = b-a$$である。

区間$$[0, k]$$上の関数$$g$$を

$$g(t) = \int_{-t}^t f(x)dx - \frac{f(-t)+4f(0)+f(t)}{6}2t$$

とおく。いま示したいのは$$\vert g(k) \vert \leq \frac{1}{2880}\Vert f^{(4)}\Vert _\infty (b-a)^5$$である。

繰り返し微分して、

$$ g’(0) = 0, g’’(0) = 0, \\ g^{(3)}(t) = -\frac{t}{3}\left[-f^{(3)}(-t) + f^{(3)}(t)\right]$$

をえる。

微積分学の基本定理から、順次、次のように計算できる。

$$\begin{align*}  \vert g^{(3)}(t)\vert &= \left\vert\frac{t}{3}\right\vert \left\vert \int_{-t}^t f^{(4)}(x)dx\right\vert \\ &\leq \frac{t}{3}\Vert f^{(4)} \Vert_\infty 2t \end{align*}$$

$$\begin{align*}  \vert g’’(t)\vert &= \left\vert g’’(0) + \int_0^t g^{(3)}(t)dt\right\vert \\ &\leq \frac{2}{3}\Vert f^{(4)} \Vert_\infty \frac{t^3}{3} \end{align*}$$

$$\begin{align*}  \vert g’(t)\vert &= \left\vert g’(0) + \int_0^t g’’(t)dt\right\vert \\ &\leq \frac{2}{9}\Vert f^{(4)} \Vert_\infty \frac{t^4}{4} \end{align*}$$

$$\begin{align*}  \vert g(t)\vert &= \left\vert g(0) + \int_0^t g’(t)dt\right\vert \\ &\leq \frac{1}{18}\Vert f^{(4)} \Vert_\infty \frac{t^5}{5} \end{align*}$$

よって、

$$\begin{align*}  \vert g(k)\vert&\leq \frac{1}{90}\Vert f^{(4)} \Vert_\infty k^5 \\&=  \frac{1}{90}\Vert f^{(4)} \Vert_\infty \left( \frac{b-a}{2}\right)^5 \\&= \frac{1}{2880}\Vert f^{(4)} \Vert_\infty (b-a)^5\end{align*}$$

となる。
複合シンプソン則の誤差評価については、複合中点則の証明と同じように、シンプソン則の誤差評価を足し合わせれば導出できる。（証明終わり）

## 2.3 ラグランジュ補間多項式

### 定義（ラグランジュ補間多項式）

区間$$[a, b]$$を

$$a\leq x_0 \leq x_1 \leq \cdots \leq x_n \leq b$$

と$$(n+1)$$個の点で$$n$$分割する。この点たちから、多項式$$L_i(x)$$を

$$L_i(x) := \frac{\prod_{j\neq i}(x-x_j)}{\prod_{j \neq i} (x_i - x_j)}$$

と定義する。
関数$$f: [a, b] \to \mathbb{R}$$に対して、多項式関数$$p(x)$$を

$$ p(x) := \sum_i f(x_i)L_i(x)$$

とおくと、

$$p(x_i) = f(x_i)$$

を満たす。この$$p$$を$$f$$のラグランジュ補間多項式という。

## 2.4 直交多項式とガウス型積分公式

### 定義（Gauss-Chebyshev積分公式）
$$f: [-1, 1] \to \mathbb{R}$$を連続関数とする。また、$$n$$を自然数とする。

$$x_i = \cos\frac{i+1/2}{n+1}\pi, \, W_i = \frac{\pi}{n+1}\, (i = 0, 1, \ldots, n)$$

として、$$G_n^c(f)$$を

$$G_n^c(f) := \sum_{i=0}^n W_i f(x_i)$$

で定める。これは積分$$\int_{-1}^1 f(x)w(x)dx$$の近似になっている。ただし、$$w(x) = 1/\sqrt{1-x^2}$$である。これをGauss-Chebyshev積分公式という。

#### 注意

$$W_i = \frac{\pi}{n+1}$$

は本来は

$$W_i = \int_{-1}^1 L_i(x) \frac{1}{\sqrt{1-x^2}} dx$$

で定義されるものである。これらが等しいこと、すなわち

$$  \int_{-1}^1 L_i(x) \frac{1}{\sqrt{1-x^2}} dx = \frac{\pi}{n+1}$$

を示すのは見た目の易しさに反して大変なようである。その計算方針は以下のとおりである。（どこかの大学の講義資料（英語）で見つけた計算方法なのだが、どこの講義資料であったかのURLを紛失してしまった。）

まず、補題として以下を示す。

##### 補題
$$n \geq 1$$に対して、以下が成り立つ。

$$ PV \int_0^\pi \frac{\cos n\theta}{\cos\theta - \cos \phi}d\theta = \pi\frac{\sin n\phi}{\sin \phi} $$

$$ PV \int_0^\pi \frac{\sin n\theta \sin\theta}{\cos\theta - \cos \phi}d\theta = -\pi  \cos n\phi$$

ここで、$$PV\int f(x)dx$$というのはコーシーの主値の意味である。すなわち、$$\theta$$が$$[0, \phi-\epsilon]$$と$$[\phi+\epsilon, \pi]$$を動く積分の、$$\epsilon \to +0$$の極限値である。

この補題を（$$n \geq 0$$に対する）帰納法で示す。そのために、

$$PV\int_0^\pi \frac{d\theta}{\cos \theta - \cos \phi} = 0$$

を、$$\cos \theta = \frac{1}{2}(\exp(i\theta) + \exp(-i\theta))$$などと、留数積分を使って計算する。

あとは、この補題をつかって、$$W_i$$を計算すればよい。$$L_i(x)$$はチェビシェフ多項式を用いて$$L_i(x) = \frac{T_{n+1}(x)/(x-x_i)}{T_{n+1}(x_i)}$$と表せることに注意する。


さて、Gauss-Chebyshev積分公式が積分の近似になっていることを示そう。

### 定理（Gauss-Chebyshev積分公式の収束）

$$G_n^c(f) \to \int_{-1}^{1}f(x)w(x)dx \, (n\to \infty)$$

#### 証明
$$\epsilon > 0$$を任意にとる。

ワイエルシュトラスの多項式近似定理（あるいはストーン・ワイエルシュトラスの定理）から、

$$\Vert f - p \Vert_\infty \leq \frac{\epsilon}{2\pi}$$

となるような多項式$$p(x)$$をとることができる。

すると、式を以下のように分解することができる。

$$ \begin{align} \left\vert G_n^c(f) - \int_{-1}^1 f(x)w(x)dx \right\vert & \leq \left\vert G_n^c(f) - G_n^c(p) \right\vert \\ &+ \left\vert G_n^c (p) - \int_{-1}^1 p(x)w(x)dx \right\vert \\ &+ \left\vert \int_{-1}^1 p(x)w(x)dx - \int_{-1}^1 f(x)w(x)dx \right\vert \end{align}$$

(1)については、

$$\begin{align*} & \vert G_N^c(f) - G_n^c(p) \vert \\=& \vert \sum W_i f(x_i) - \sum W_i p(x_i)\vert \\\leq& \sum W_i \vert f(x_i)- p(x_i) \vert \\=& (n+1)\frac{\pi}{n+1}\frac{\epsilon}{2\pi} \\=&\frac{\epsilon}{2}\end{align*}$$

と評価でき、
(2)については、ラグランジュ補間多項式の性質から、

$$\begin{align*} G_n^c(p) &= \sum W_i p(x_i) \\&= \int_{-1}^1 \left( \sum L_i(x) p(x_i)\right) w(x)dx \\&= \int_{-1}^1 p(x)w(x)dx \, (n >> 0)\end{align*}$$

と計算できる。したがって、$$n$$を十分大きくとれば(2)は$$0$$になる。

(3)については、

$$\begin{align*} & \left\vert \int_{-1}^1 f(x)w(x)dx - \int_{-1}^1 p(x)w(x)dx \right \vert \\ \leq & \Vert f-p \Vert _ \infty \int_{-1}^1 w(x)dx \\ =& \frac{\epsilon}{2}\end{align*}$$

と計算できる。

したがって、$$n$$が十分大きければ、$$(1)+(2)+(3) \leq \epsilon$$となる。（証明終わり）

## 参考文献

* 齊藤宣一『数値解析』
* 齊藤宣一『数値解析入門』
* どこかの大学の講義資料（失念）

