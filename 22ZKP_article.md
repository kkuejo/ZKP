---
title: "Reed-Solomon 符号と FFT — STARK の数学的基盤"
emoji: "🎶"
type: "tech"
topics:
  - zkp
  - stark
  - reedsolomon
  - fft
  - ntt
published: false
---

**日付**: 2026年4月22日
**学習内容**: STARK の理論は **Reed-Solomon 符号** と **FFT (Fast Fourier Transform)** の上に立つ。本記事では **(1) 誤り訂正符号の基礎**、**(2) Reed-Solomon 符号の定義**、**(3) 最小距離と誤り訂正能力**、**(4) FFT の原理**、**(5) 有限体上の FFT (NTT)**、**(6) ブローアップと STARK での役割**、**(7) 近接性 (proximity) の概念** を扱う。式の展開を丁寧に行い、FFT の $O(n \log n)$ がどう導出されるかも示す。次の記事の FRI の前提知識。

## 0. 本記事の位置づけ

Article 21 で「STARK は多項式を低次数テストする」と述べた。その中身は:

1. **多項式 $f(X)$ を評価点の集合で評価**（Reed-Solomon 符号化）
2. **評価値が多項式 $f$ と本当に整合的か** を確率的にテスト（Low-Degree Test）

本記事は (1) の背景を固める。FFT で高速化されない STARK は実用にならないので、この 2 つは STARK の計算基盤。

構成:

- **第1章**: 誤り訂正符号の基礎
- **第2章**: Reed-Solomon 符号
- **第3章**: 最小距離
- **第4章**: FFT
- **第5章**: NTT (Number Theoretic Transform)
- **第6章**: ブローアップと STARK での役割
- **第7章**: 近接性
- **第8章**: Q&A とまとめ

## 1. 誤り訂正符号の基礎

### 1.1 問題設定

通信路にノイズがある状況:

```
送信側: [データ m] → [符号化 c] → [通信路] → [受信 c' = c + e] → [復号] → [m]
```

ノイズ $e$ で一部のビットが変わっても、元のデータ $m$ を復元したい。

### 1.2 線形符号 (linear code)

**線形符号** $C$ は $\mathbb{F}_q^n$ の部分ベクトル空間:

- $\mathbb{F}_q^k$ のメッセージを $\mathbb{F}_q^n$ の符号語に埋め込む
- 次元 $k$、長さ $n$、レート $R = k/n$
- **生成行列 $G$**: $c = m G$ で符号化

### 1.3 Hamming 距離

$c_1, c_2 \in \mathbb{F}_q^n$ に対して、異なる位置の数:

$$
d(c_1, c_2) := |\{i : c_{1,i} \neq c_{2,i}\}|
$$

**最小距離** $d_{\min} = \min_{c_1 \neq c_2 \in C} d(c_1, c_2)$。

### 1.4 誤り訂正能力

**定理**: 最小距離 $d_{\min}$ の符号は、$\lfloor (d_{\min} - 1)/2 \rfloor$ 個までの誤りを訂正できる。

**証明**: 各符号語を中心とした半径 $\lfloor (d_{\min}-1)/2 \rfloor$ の Hamming 球は互いに交わらない。受信 $c'$ が符号語 $c$ から $t \leq \lfloor (d_{\min}-1)/2 \rfloor$ の距離にあれば、最近の符号語が $c$ で一意に決まる。$\square$

### 1.5 Singleton bound

**定理**: $[n, k, d]$ 符号では $d \leq n - k + 1$。

等号成立する符号を **MDS (Maximum Distance Separable)** 符号という。Reed-Solomon 符号は MDS。

## 2. Reed-Solomon 符号

### 2.1 定義

$\mathbb{F}_q$ 上の **Reed-Solomon 符号** は:

- パラメータ: 長さ $n \leq q$、次元 $k$
- 評価点 $\alpha_1, \alpha_2, \ldots, \alpha_n \in \mathbb{F}_q$ を決める（相異なる）
- メッセージ $m = (m_0, m_1, \ldots, m_{k-1}) \in \mathbb{F}_q^k$ を多項式と見なす:

$$
f_m(X) = m_0 + m_1 X + \cdots + m_{k-1} X^{k-1}
$$

- 符号語: $c = (f_m(\alpha_1), f_m(\alpha_2), \ldots, f_m(\alpha_n))$

つまり **「次数 $k-1$ 以下の多項式の $n$ 点評価」**が符号語。

### 2.2 最小距離

**定理**: Reed-Solomon 符号の最小距離は $d = n - k + 1$。

**証明**: 2 つの異なる符号語 $c^{(1)}, c^{(2)}$ は 2 つの異なる次数 $k-1$ 以下の多項式 $f_1, f_2$ から来る。$f_1 - f_2$ は非ゼロ次数 $k-1$ 以下の多項式なので、根の数は $\leq k - 1$。

したがって $c^{(1)}_i = c^{(2)}_i$ となる $i$ の数 $\leq k - 1$ → $d(c^{(1)}, c^{(2)}) \geq n - (k - 1) = n - k + 1$。$\square$

RS 符号は Singleton bound に一致 → **MDS**。

### 2.3 デコード

- **Berlekamp-Massey algorithm**: 誤り位置多項式を計算
- **Welch-Berlekamp algorithm**: 誤り訂正 + 列探索
- **Forney algorithm**: 誤りの値を算出

実用では RS 符号は CD, QR コード, DVD, 衛星通信で使われる。

### 2.4 ZKP での役割

ZKP では「誤り訂正」そのものより、**「符号語間の距離」という性質**を使う:

- 正しい多項式 $f$ は符号語
- 偽の多項式 $f'$ (高次) は符号語ではない（符号から遠い）

この距離を確率的に検出するのが **FRI** (Article 23)。

## 3. 最小距離の活用

### 3.1 Relative Distance

$\delta = d / n$ を**相対距離**と呼ぶ。レート $R = k/n$ と相対距離 $\delta$ のトレードオフ:

$$
R + \delta \leq 1 \quad (\text{Singleton bound})
$$

RS 符号は $R + \delta = 1 + 1/n \approx 1$。

### 3.2 STARK での典型的な設定

- $\rho = k/n$ を**レート**（または逆で $\rho = n/k$ とも書かれる、文脈で判断）
- 典型: $\rho = 1/2, 1/4, 1/8$（ブローアップ $\rho^{-1} = 2, 4, 8$）

ブローアップが大きいほど誤り訂正能力が高い = 健全性が強い。ただし Prover / Verifier コストも増える。

## 4. FFT

### 4.1 問題

多項式 $f(X) = \sum_{i=0}^{n-1} a_i X^i$ を $n$ 点 $\alpha_0, \ldots, \alpha_{n-1}$ で評価する。

素朴には各点 $O(n)$ × $n$ 点 = $O(n^2)$。**FFT** は $O(n \log n)$ で行う。

### 4.2 DFT (Discrete Fourier Transform)

$\omega$ を 1 の $n$ 乗根（$\omega^n = 1$, $\omega^k \neq 1$ for $0 < k < n$）とし、評価点を:

$$
\alpha_i = \omega^i
$$

とする。これが **DFT**。

DFT の行列表現:

$$
\begin{pmatrix}
f(1) \\
f(\omega) \\
f(\omega^2) \\
\vdots
\end{pmatrix}
= W \begin{pmatrix}
a_0 \\
a_1 \\
a_2 \\
\vdots
\end{pmatrix}, \quad
W_{ij} = \omega^{ij}
$$

### 4.3 FFT のアイデア

$n$ が 2 のべきとする。$f$ を偶次と奇次に分割:

$$
f(X) = f_e(X^2) + X f_o(X^2)
$$

- $f_e$: 偶次係数
- $f_o$: 奇次係数
- どちらも次数 $n/2 - 1$ 以下

### 4.4 再帰的構造

$f(\omega^i)$ と $f(-\omega^i) = f(\omega^{i + n/2})$ を一緒に計算:

$$
f(\omega^i) = f_e(\omega^{2i}) + \omega^i f_o(\omega^{2i})
$$

$$
f(\omega^{i + n/2}) = f_e(\omega^{2i}) - \omega^i f_o(\omega^{2i})
$$

1 ペア計算 = 1 加算 + 1 減算 + 1 乗算。$n/2$ ペアで $n/2$ 個の evenと $n/2$ 個の odd を合成。

### 4.5 時間計算量

再帰: $T(n) = 2 T(n/2) + O(n)$

Master theorem: $T(n) = O(n \log n)$。

### 4.6 数値例: $n = 4$

$f(X) = a_0 + a_1 X + a_2 X^2 + a_3 X^3$

$f_e(Y) = a_0 + a_2 Y, f_o(Y) = a_1 + a_3 Y$

$\omega = i$ (複素 4 次単位根)、$\omega^2 = -1$。

計算:

- $f_e(1) = a_0 + a_2, f_e(-1) = a_0 - a_2$
- $f_o(1) = a_1 + a_3, f_o(-1) = a_1 - a_3$

最終:

- $f(1) = (a_0+a_2) + 1 \cdot (a_1+a_3) = a_0+a_1+a_2+a_3$
- $f(i) = (a_0-a_2) + i \cdot (a_1-a_3)$
- $f(-1) = (a_0+a_2) - 1 \cdot (a_1+a_3)$
- $f(-i) = (a_0-a_2) - i \cdot (a_1-a_3)$

## 5. NTT (Number Theoretic Transform)

### 5.1 有限体版 FFT

実数 FFT は複素数の単位根を使うが、ZKP では**有限体上**で同じ仕組みが必要。これが **NTT (Number Theoretic Transform)**。

### 5.2 要件

$\mathbb{F}_p$ 上で FFT 可能にするには:

- **$p - 1$ が $n$ で割り切れる** → 1 の $n$ 乗根が $\mathbb{F}_p$ に存在

### 5.3 Goldilocks 素数

$p = 2^{64} - 2^{32} + 1$ は $p - 1 = 2^{64} - 2^{32} = 2^{32}(2^{32} - 1)$。

$2^{32}$ で割れる → 長さ $2^{32}$ までの NTT 可能。Plonky2 はこれを使用。

### 5.4 BN254 / BLS12-381

これらのスカラー体 $\mathbb{F}_r$ も $r - 1$ が $2^{28}$ や $2^{32}$ で割れるよう設計。Groth16, PLONK の FFT で使われる。

### 5.5 NTT の実装

基本は FFT と同じ。ただし:

- 複素乗算なし、すべて $\mathbb{F}_p$ での乗算
- モジュラー逆元が時々必要
- **Inverse NTT** は前進 NTT + 定数倍（$n^{-1}$ を掛ける）

### 5.6 GPU / FPGA での高速化

NTT は並列化しやすく、GPU で 10〜100 倍高速化可能。大規模 SNARK / STARK 証明の Prover では NTT 高速化が性能の鍵。

## 6. ブローアップと STARK での役割

### 6.1 トレース評価ドメイン

STARK の trace は長さ $n$。これを長さ $n$ の単位根集合 $H$ 上で補間。

### 6.2 拡張ドメイン

STARK は**より大きなドメイン** $L$ で評価する:

$$
|L| = \rho^{-1} \cdot n
$$

たとえば $\rho = 1/4$ なら $|L| = 4n$。

### 6.3 拡張ドメインを使う理由

- **誤り訂正符号** としての余裕を持たせる
- 多項式の次数が $n$ までなので、評価点 $4n$ のうち多くても $n$ 個までしか同じ値にならない
- つまり **相対距離 $\geq 1 - \rho = 3/4$**

### 6.4 Merkle コミットメント

$L$ 上の $4n$ 個の評価値を Merkle ツリーに格納。Verifier がランダム位置を選んで確認。

ブローアップが大きいほど、ランダム位置を少し選ぶだけで確率的安全性を得る。

## 7. 近接性 (Proximity)

### 7.1 Close or Far

多項式 $f$ と値の列 $u = (u_0, u_1, \ldots, u_{N-1})$ について:

$$
\delta(f, u) = \frac{|\{i : f(\alpha_i) \neq u_i\}|}{N}
$$

**$u$ が符号語 $\{f(\alpha_i)\}$ に $\delta$-close** とは、上の距離が $\leq \delta$。

### 7.2 Low-Degree Test の問題

Verifier が Merkle root $C_u$ を受け取り、「$u$ は**ある次数 $k-1$ 以下の多項式**の評価と**近い**か」を確認したい。

### 7.3 STARK での使い方

Prover は「$u$ は正しい低次多項式の評価」と主張。しかし万一 $u$ が高次だったら、Verifier はそれを検出する必要がある。

**FRI** (次記事) はこれを $O(\log^2 n)$ で行う。

### 7.4 Unique Decoding vs List Decoding

- **Unique decoding radius**: $\delta \leq (1 - \rho)/2$。この範囲なら受信語から符号語が一意に復号
- **List decoding radius**: $\delta \leq 1 - \sqrt{\rho}$（Johnson bound）。この範囲なら**少数の候補**に絞れる

STARK の健全性は **List decoding radius** で議論される。

## 8. Q&A

### Q1: Reed-Solomon 符号のビットレート $R = k/n$ と STARK の $\rho$ の関係？

STARK のブローアップ $\rho^{-1}$ は「RS 符号のレートの逆」$1/R$。つまり $\rho = R$。

### Q2: FFT で「$n$ が 2 のべき」でない場合は？

Bluestein アルゴリズムや Rader アルゴリズムで任意の $n$ に対応可能。ただし遅い。STARK では 2 のべきに**パディング**するのが普通。

### Q3: NTT の計算量は？

$O(n \log n)$ 乗算 + $O(n \log n)$ 加算。有限体乗算が高速なら、数百万項の NTT が数秒で実行可能。

### Q4: Binius の bit-level アプローチは？

Binius は $\mathbb{F}_{2^k}$ を使う STARK で、乗算が XOR/AND に還元される。**さらに高速な NTT**が可能。

### Q5: Reed-Solomon 以外の符号は使える？

使える。Ligero / Brakedown は **線形時間エンコード可能な符号** (e.g., Spielman codes) を使う。RS 符号は $O(n \log n)$ だが線形符号は $O(n)$、Prover が更に高速。

### Q6: FFT が必要ない ZKP は？

Spartan や HyperPlonk は MLE ベースで FFT 不要。これらは Prover が線形時間。

## 9. まとめ

### 本記事の要点

1. **Reed-Solomon 符号**: 次数 $k-1$ 以下の多項式の $n$ 点評価
2. **最小距離** $d = n - k + 1$（MDS）
3. **FFT**: $O(n \log n)$ で多項式評価
4. **NTT**: 有限体上の FFT、$p - 1$ が $n$ で割れる必要
5. **ブローアップ**: 拡張ドメインで誤り訂正余裕
6. **近接性**: 符号との距離、低次数テストの基盤
7. **実装**: Plonky2, Halo2, STARK が NTT/FFT を多用、GPU 高速化可能

### 次の記事（Article 23）へ

次の記事は **FRI (Fast Reed-Solomon IOP of Proximity)**。Reed-Solomon 符号の低次数性を $O(\log^2 n)$ で証明する STARK の心臓部。多項式を半分に折る「folding」操作を $\log n$ 回繰り返す仕組み。

### 3行サマリ

- **Reed-Solomon** = 多項式を $n$ 点で評価、MDS 符号
- **FFT/NTT で $O(n \log n)$** の高速変換
- **ブローアップ** で STARK の健全性に余裕、拡張ドメインで Merkle コミット

---

## 参考文献

- Shu Lin, Daniel J. Costello. *Error Control Coding, 2nd ed.* Prentice Hall, 2004.
- Richard E. Blahut. *Theory and Practice of Error Control Codes.* Addison-Wesley, 1983.
- T. H. Cormen et al. *Introduction to Algorithms, 3rd ed.* Chapter 30 (FFT), 2009.
- Eli Ben-Sasson et al. *Fast Reed-Solomon Interactive Oracle Proofs of Proximity.* ICALP 2018.
- Polygon Zero. *Plonky2: NTT over Goldilocks.* 2022.
