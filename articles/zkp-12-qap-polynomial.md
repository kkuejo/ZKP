---
title: "[ゼロ知識証明入門シリーズ 12/33] QAP — 制約を多項式に昇華する"
emoji: "🎼"
type: "tech"
topics:
  - zkp
  - snark
  - qap
  - polynomial
  - groth16
published: false
---

**日付**: 2026年4月22日
**学習内容**: **QAP (Quadratic Arithmetic Program)** は、R1CS を多項式の等式に変換する仕組み。「たくさんの制約を満たすか」を「**1つの多項式がある多項式で割り切れるか**」というコンパクトな問いに変える。これが Groth16 SNARK の核。本記事では **(1) QAP の動機**、**(2) R1CS → QAP の変換手順**、**(3) ラグランジュ補間の役割**、**(4) 目標多項式 $Z(X)$ と消滅多項式**、**(5) QAP の成立条件** $P(X) = Z(X) H(X)$、**(6) 具体例の完全展開**、**(7) QAP の拡張 (QSP, SSP, GKR)** を扱う。式の展開を丁寧に追う。

## 0. 本記事の位置づけ

Article 11 で R1CS は「$m$ 個の制約」という**離散的**な構造だった。これをそのままプロトコル化すると、Prover は $m$ 個の式を全部 Verifier に送る必要があり、証明が大きい。

QAP の発想:

> **$m$ 個の制約を、1 つの多項式等式に圧縮する**

具体的には、ラグランジュ補間で $m$ 点の制約を通る多項式を作り、「**その多項式がすべての評価点でゼロ**」を「**別の多項式で割り切れる**」に変換する。この変換で証明が $O(1)$ サイズになる (Schwartz-Zippel との組み合わせで)。

構成:

- **第1章**: QAP の動機
- **第2章**: R1CS → QAP の基本ステップ
- **第3章**: ラグランジュ補間で行列列を多項式に
- **第4章**: 目標多項式 $Z(X)$
- **第5章**: QAP の成立条件
- **第6章**: 具体例の完全展開
- **第7章**: QAP の拡張
- **第8章**: Q&A とまとめ

## 1. QAP の動機

### 1.1 R1CS の表現コスト

R1CS は以下の形:

$$
\forall i \in \{1, \ldots, m\}: (A_i \vec{w})(B_i \vec{w}) = (C_i \vec{w})
$$

$m$ 個の式。これをそのまま Verify するなら、Verifier は $m$ 個の等式をチェックする必要があり、$m$ が巨大だと辛い。

### 1.2 発想: 1つの多項式等式にまとめる

各制約を「ある多項式の1点評価」と解釈:

- 制約 $i$ $\Leftrightarrow$ 多項式 $F(X)$ が $X = i$ でゼロ
- すべての制約 $\Leftrightarrow$ $F$ が $\{1, 2, \ldots, m\}$ すべてでゼロ

さらに、**$F$ が $m$ 個の根を持つ $\Leftrightarrow$ $F$ が $Z(X) = \prod_i (X - i)$ で割り切れる**。

$$
\forall i \in \{1, \ldots, m\}: F(i) = 0 \iff \exists H: F(X) = Z(X) \cdot H(X)
$$

これで「$m$ 個の制約」が「**1つの多項式除算**」に化ける。

### 1.3 QAP の契約

Prover は Witness $\vec{w}$ から多項式 $F(X), H(X)$ を構成し、等式 $F = ZH$ を示す。Verifier はランダムな $X = r$ で両辺を評価し、$F(r) = Z(r) H(r)$ を確認する（Schwartz-Zippel）。

```mermaid
flowchart LR
    R1CS[R1CS<br/>m 制約]
    QAP[QAP<br/>F(X) = Z(X) H(X)]
    Eval[ランダム点 r<br/>で評価]
    Check[F(r) = Z(r) H(r)?]
    
    R1CS --> QAP
    QAP --> Eval
    Eval --> Check
```

## 2. R1CS → QAP の基本ステップ

### 2.1 出発点

R1CS: 行列 $A, B, C \in \mathbb{F}^{m \times (n+1)}$、witness $\vec{w} \in \mathbb{F}^{n+1}$。

### 2.2 変換アイデア

**行列の各列を多項式に変換する**:

- $A$ の第 $j$ 列を**関数**「制約 $i \to A_{i,j}$」として見る
- これを $m$ 点 $(1, A_{1,j}), (2, A_{2,j}), \ldots, (m, A_{m,j})$ と見なし、ラグランジュ補間で多項式 $a_j(X)$ を作る
- $a_j(i) = A_{i,j}$ を満たす、次数 $m-1$ 以下の多項式

同様に $b_j(X)$（$B$ の列）、$c_j(X)$（$C$ の列）。

### 2.3 多項式による制約の符号化

Witness $\vec{w} = (w_0, w_1, \ldots, w_n)$ を使って、3つの線形結合多項式を定義:

$$
A(X) := \sum_{j=0}^{n} w_j \cdot a_j(X)
$$

$$
B(X) := \sum_{j=0}^{n} w_j \cdot b_j(X)
$$

$$
C(X) := \sum_{j=0}^{n} w_j \cdot c_j(X)
$$

### 2.4 重要な事実

$X = i$ を代入すると:

$$
A(i) = \sum_j w_j \cdot a_j(i) = \sum_j w_j \cdot A_{i,j} = (A\vec{w})_i
$$

同様に:

$$
B(i) = (B\vec{w})_i, \quad C(i) = (C\vec{w})_i
$$

したがって、R1CS 制約 $(A\vec{w})_i (B\vec{w})_i = (C\vec{w})_i$ は:

$$
A(i) \cdot B(i) = C(i)
$$

多項式の言葉で書くと:

$$
A(X) B(X) - C(X) = 0 \quad \text{for all } X \in \{1, \ldots, m\}
$$

### 2.5 割り切れ条件への変換

多項式 $A(X)B(X) - C(X)$ が $\{1, \ldots, m\}$ すべてでゼロ $\iff$ $Z(X) := \prod_{i=1}^{m}(X - i)$ で割り切れる。

したがって **QAP の成立条件**:

$$
\boxed{A(X) B(X) - C(X) = Z(X) \cdot H(X)}
$$

ここで $H(X)$ は商多項式（Prover が計算）。

## 3. ラグランジュ補間で行列列を多項式に

### 3.1 ラグランジュの復習

$m$ 個の異なる点 $x_1, \ldots, x_m$ と値 $y_1, \ldots, y_m$ について、次数 $m-1$ 以下で $(x_i, y_i)$ を通る多項式は一意で:

$$
P(X) = \sum_{i=1}^{m} y_i \cdot L_i(X), \quad L_i(X) = \prod_{j \neq i} \frac{X - x_j}{x_i - x_j}
$$

### 3.2 列ごとの補間

行列 $A$ の第 $j$ 列を $\{A_{1,j}, A_{2,j}, \ldots, A_{m,j}\}$。これをラグランジュ補間:

$$
a_j(X) = \sum_{i=1}^{m} A_{i,j} \cdot L_i(X)
$$

同様に $b_j(X), c_j(X)$。

### 3.3 計算コスト

- 列数: $n + 1$
- 各列の補間: $O(m^2)$ 素朴、$O(m \log m)$ FFT

通常はむしろ、**$a_j, b_j, c_j$ を評価点で保持**し、点→点の形で計算する（FFT と相性が良い）。

### 3.4 構造の可視化

```mermaid
flowchart TD
    Matrix[行列 A<br/>m × (n+1)]
    Col[列 j を取り出す<br/>m 個の値]
    Lagrange[ラグランジュ補間]
    Poly["多項式 a_j(X)<br/>次数 m-1 以下"]
    
    Matrix --> Col
    Col --> Lagrange
    Lagrange --> Poly
```

## 4. 目標多項式 $Z(X)$

### 4.1 定義

$Z(X) = (X - 1)(X - 2) \cdots (X - m)$

次数 $m$、主係数 1。評価点 $\{1, 2, \ldots, m\}$ すべてで 0、他では 0 でない。

### 4.2 なぜ「消滅多項式 (vanishing polynomial)」と呼ぶか

評価点の集合 $H = \{1, 2, \ldots, m\}$ すべてでゼロになるので、$H$ の上で「消える」多項式。

### 4.3 実用では単位根を使う

実装上は $H = \{1, 2, \ldots, m\}$ よりも **1 の $m$ 乗根** $\{1, \omega, \omega^2, \ldots, \omega^{m-1}\}$ を使うことが多い。これだと:

$$
Z(X) = X^m - 1
$$

$Z(X)$ と $H$ における評価が**非常に高速** (FFT が使える)。

Article 23 で詳述。

## 5. QAP の成立条件

### 5.1 Prover の作業

Prover は Witness $\vec{w}$ が R1CS を満たすと主張する。Prover は:

1. $A(X), B(X), C(X)$ を計算
2. $P(X) := A(X) B(X) - C(X)$ を計算
3. $H(X) := P(X) / Z(X)$ を計算 (割り切れるはず)
4. $A, B, C, H$ のいずれか、または多項式コミットメントを Verifier に渡す

### 5.2 Verifier の作業

Verifier は:

1. ランダム点 $r$ を選ぶ (または CRS から暗黙の $\tau$ を使う)
2. $A(r), B(r), C(r), H(r), Z(r)$ を評価 (コミットメント経由)
3. 等式 $A(r) B(r) - C(r) = Z(r) H(r)$ を確認

### 5.3 健全性

もし Prover の $\vec{w}$ が R1CS を満たさないと、$P(X)$ は $Z(X)$ で割り切れない。このとき:

$$
P(X) - Z(X) H(X) \neq 0 \quad (\text{どんな } H \text{ をとっても})
$$

この非ゼロ多項式がランダム点 $r$ でゼロになる確率は、Schwartz-Zippel により $\leq d/|\mathbb{F}|$。典型的に $2^{-200}$ 以下。

### 5.4 ゼロ知識化

単にこの等式を送るだけでは witness の情報が漏れる。そこで Prover は:

- $A(X), B(X), C(X)$ をコミットメントで送る（値そのものは隠す）
- ランダムなマスキング多項式を加えて分布をランダム化

詳細は Article 16 (Groth16)。

## 6. 具体例の完全展開

### 6.1 例: $y = x^3 + x + 5$ の QAP 化

Article 11 で作った R1CS:

- 変数: $\vec{w} = (w_0, w_1, w_2, w_3, w_4) = (1, x, y, t_1, t_2)$
- 制約:
  - C1: $(x)(x) = (t_1)$
  - C2: $(t_1)(x) = (t_2)$
  - C3: $(5 + x + t_2)(1) = (y)$

### 6.2 行列 $A, B, C$

$$
A = \begin{pmatrix}
0 & 1 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 & 0 \\
5 & 1 & 0 & 0 & 1
\end{pmatrix}, \quad
B = \begin{pmatrix}
0 & 1 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 & 0 \\
1 & 0 & 0 & 0 & 0
\end{pmatrix}, \quad
C = \begin{pmatrix}
0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 1 \\
0 & 0 & 1 & 0 & 0
\end{pmatrix}
$$

### 6.3 列ごとのラグランジュ補間

$m = 3$、評価点は $\{1, 2, 3\}$ とする。

ラグランジュ基底:

$$
L_1(X) = \frac{(X-2)(X-3)}{(1-2)(1-3)} = \frac{(X-2)(X-3)}{2}
$$

$$
L_2(X) = \frac{(X-1)(X-3)}{(2-1)(2-3)} = -(X-1)(X-3)
$$

$$
L_3(X) = \frac{(X-1)(X-2)}{(3-1)(3-2)} = \frac{(X-1)(X-2)}{2}
$$

### 6.4 $A$ の列 0 の補間 (列の値 = $(0, 0, 5)$)

$$
a_0(X) = 0 \cdot L_1 + 0 \cdot L_2 + 5 \cdot L_3 = 5 \cdot \frac{(X-1)(X-2)}{2}
$$

展開: $a_0(X) = \frac{5}{2}(X^2 - 3X + 2) = \frac{5}{2}X^2 - \frac{15}{2}X + 5$

**確認**: $a_0(1) = 0$（$L_3(1) = 0$）、$a_0(2) = 0$、$a_0(3) = 5$。◯

### 6.5 $A$ の列 1 ($x$ の係数、値 $(1, 0, 1)$)

$$
a_1(X) = 1 \cdot L_1 + 0 \cdot L_2 + 1 \cdot L_3
$$

$$
= \frac{(X-2)(X-3)}{2} + \frac{(X-1)(X-2)}{2}
$$

$$
= \frac{(X-2)[(X-3) + (X-1)]}{2} = \frac{(X-2)(2X-4)}{2} = (X-2)^2
$$

**確認**: $a_1(1) = 1, a_1(2) = 0, a_1(3) = 1$。◯

### 6.6 $A(X)$ の構築

ここで Witness の具体的な値を使う。たとえば $x = 3$ を代入して $y = 3^3 + 3 + 5 = 35$。

- $t_1 = 9, t_2 = 27$
- $\vec{w} = (1, 3, 35, 9, 27)$

$$
A(X) = 1 \cdot a_0(X) + 3 \cdot a_1(X) + 35 \cdot a_2(X) + 9 \cdot a_3(X) + 27 \cdot a_4(X)
$$

各 $a_j(X)$ を計算する必要があるが、**$A(i)$ は直接 R1CS から計算できる**:

- $A(1) = 0 + 3 \cdot 1 + 0 + 0 + 0 = 3$
- $A(2) = 0 + 0 + 0 + 9 \cdot 1 + 0 = 9$
- $A(3) = 5 + 3 \cdot 1 + 0 + 0 + 27 \cdot 1 = 35$

これらの評価値 $(3, 9, 35)$ を通るラグランジュ補間で $A(X)$ が求まる。

同様に $B(X), C(X)$:

- $B$: $(3, 3, 1)$
- $C$: $(9, 27, 35)$

### 6.7 等式の検証

評価点 $X = 1$ で: $A(1)B(1) = 3 \cdot 3 = 9 = C(1)$ ◯  
$X = 2$: $9 \cdot 3 = 27 = C(2)$ ◯  
$X = 3$: $35 \cdot 1 = 35 = C(3)$ ◯

### 6.8 $Z(X)$ と $H(X)$

$Z(X) = (X-1)(X-2)(X-3)$、次数 3。

$P(X) = A(X)B(X) - C(X)$ の次数は $(m-1) + (m-1) = 4$。$Z$ で割ると $H$ の次数は $4 - 3 = 1$。

$P(1) = P(2) = P(3) = 0$ を確認済みなので、因数定理で $P(X) = Z(X) H(X)$ の形。$H(X)$ は1次多項式として一意に決まる。

### 6.9 もしも偽の witness だったら

たとえば $\vec{w}' = (1, 3, 35, 9, 28)$（$t_2 = 28$ と誤る）なら:

- C2 の制約: $t_1 \cdot x = t_2$ → $9 \cdot 3 = 27 \neq 28$ で失敗

$A(2) B(2) - C(2) = 9 \cdot 3 - 28 = -1 \neq 0$。したがって $P(X)$ は $X = 2$ でゼロにならない → $Z(X)$ で割り切れない → 等式が成立しない。Prover は偽の $H(X)$ を作れない。

## 7. QAP の拡張

### 7.1 QSP (Quadratic Span Programs)

QAP の前身。今はほぼ QAP に置き換わった。

### 7.2 SSP (Square Span Programs)

論理回路（AND/OR/NOT）専用の最適化版。QAP より効率的な場合がある。

### 7.3 MLE ベースの制約（Sumcheck）

Spartan や HyperPlonk では、ラグランジュ補間の代わりに **Multilinear Extension (MLE)** を使う (Article 14)。

### 7.4 AIR (STARK)

実行トレースの行を多項式化し、**隣接行の遷移制約** を多項式で書く。QAP とは発想が違うが、「制約を多項式にする」という哲学は共通。

## 8. Q&A

### Q1: 評価点 $\{1, 2, \ldots, m\}$ と単位根、どちらを使う？

**実用では単位根**。$Z(X) = X^m - 1$ が単純で、FFT で高速評価できる。$\{1, 2, 3, \ldots\}$ は教科書的な説明用。

### Q2: 多項式の次数はどれくらい？

- $a_j(X), b_j(X), c_j(X)$: 次数 $m - 1$
- $A(X), B(X), C(X)$: 次数 $m - 1$
- $A(X) B(X) - C(X)$: 次数 $2(m - 1)$
- $Z(X)$: 次数 $m$
- $H(X)$: 次数 $m - 2$

### Q3: QAP は witness を完全に隠す？

**生の QAP 式は隠さない**。$A(X), B(X)$ などを平文で送ると witness が漏れる。そこで **コミットメントで送り、かつランダムマスキング**を加える（Groth16 の設計）。

### Q4: ラグランジュ補間の計算量は？

- 係数表現: $O(m^2)$
- 評価表現: $O(m)$ で持てる、評価は $O(1)$
- 係数 ↔ 評価の変換: $O(m \log m)$ で FFT

実用では評価表現で保持し、必要に応じて IFFT で係数に戻す。

### Q5: Prover のコストは？

ほぼ FFT コスト $O(m \log m)$ + コミットメントコスト $O(m)$。100万制約なら 数秒〜数十秒。

### Q6: Verifier のコストは？

Groth16 では **$O(|\text{public input}|)$**、ほぼ $|\text{public input}|$ 回の楕円曲線演算 + 3 回のペアリング。総検証時間 $\sim$ 数ミリ秒。

## 9. まとめ

### 本記事の要点

1. **QAP**: R1CS を多項式等式 $A(X) B(X) - C(X) = Z(X) H(X)$ に変換
2. 行列列をラグランジュ補間で多項式化: $a_j, b_j, c_j$
3. Witness の線形結合で $A(X), B(X), C(X)$ を得る
4. 目標多項式 $Z(X) = \prod (X - i)$ (実用では $X^m - 1$)
5. 制約が成立 ⟺ $Z(X)$ で割り切れる
6. Schwartz-Zippel でランダム点評価の健全性を得る
7. 拡張: QSP, SSP, MLE, AIR

### 次の記事（Article 13）へ

次の記事では、QAP の多項式を**コミット**し、**ランダム点で評価**するための道具 **多項式コミットメント (PCS)** を扱う。特に **KZG コミットメント** を詳細に導出する。ペアリングを駆使した美しい構造。

### 3行サマリ

- **QAP = R1CS の多項式版、$A \cdot B - C = Z \cdot H$ に圧縮**
- **$m$ 個の制約を1本の多項式等式に**、Schwartz-Zippel で健全性担保
- **Groth16 の核**、実装の重さはほぼ FFT コスト

---

## 参考文献

- Rosario Gennaro, Craig Gentry, Bryan Parno, Mariana Raykova. *Quadratic Span Programs and Succinct NIZKs without PCPs.* EUROCRYPT 2013.
- Vitalik Buterin. *Quadratic Arithmetic Programs: from Zero to Hero.* Medium, 2016.
- Maksym Petkus. *Why and How zk-SNARK Works.* 2019.
- ZKP MOOC Lecture 9 (UC Berkeley, 2023).
