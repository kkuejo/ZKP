---
title: "[ゼロ知識証明入門シリーズ 11/30] R1CS — 制約システムの標準形"
emoji: "🧮"
type: "tech"
topics:
  - zkp
  - snark
  - r1cs
  - circuit
  - constraint
published: false
---

**日付**: 2026年4月22日
**学習内容**: Article 10 で算術回路を学んだ。本記事ではその回路を **R1CS (Rank-1 Constraint System)** という統一形式に翻訳する。R1CS は「**3つのベクトルの内積について $A \cdot w \odot B \cdot w = C \cdot w$ が成立する**」という形の制約の集合で、Groth16 や多くの SNARK ツール（Circom, Arkworks, libsnark）の**標準入力形式**。本記事では **(1) R1CS の定義**、**(2) 回路 → R1CS 変換の具体手順**、**(3) Witness の定義**、**(4) 行列表現**、**(5) 具体例の完全展開**、**(6) R1CS の限界と Plonkish への進化** を扱う。次の記事の QAP の前提知識。

## 0. 本記事の位置づけ

算術回路は直感的だが、**ZKP プロトコルで直接扱うにはまだ自由度が高すぎる**。ゲートの種類、配線、定数の扱いが一意でない。そこで R1CS という「**すべての制約を1つの標準形式にまとめた表現**」が必要になる。

R1CS の形は次の通り:

$$
(\vec{a}_i \cdot \vec{w}) \times (\vec{b}_i \cdot \vec{w}) = (\vec{c}_i \cdot \vec{w})
$$

これが $m$ 個並ぶ（$i = 1, \ldots, m$）。ここで $\vec{w}$ は全信号を並べたベクトル（**witness**）で、$\vec{a}_i, \vec{b}_i, \vec{c}_i$ は回路から決まる係数ベクトル。

構成:

- **第1章**: R1CS の定義
- **第2章**: 回路 → R1CS 変換の手順
- **第3章**: Witness ベクトルの構造
- **第4章**: 行列表現
- **第5章**: 具体例の完全展開
- **第6章**: R1CS の拡張・限界
- **第7章**: Q&A とまとめ

## 1. R1CS の定義

### 1.1 Rank-1 Constraint System

**R1CS** は以下のような**制約の集合**:

$$
\forall i \in \{1, \ldots, m\}: \quad (\vec{a}_i \cdot \vec{w})(\vec{b}_i \cdot \vec{w}) = (\vec{c}_i \cdot \vec{w})
$$

- $\vec{w} \in \mathbb{F}_p^{n+1}$: Witness ベクトル（1, 公開入力, 中間変数）
- $\vec{a}_i, \vec{b}_i, \vec{c}_i \in \mathbb{F}_p^{n+1}$: 制約ごとの係数ベクトル
- $\cdot$ は内積

### 1.2 「Rank-1」の意味

各制約は **左辺 = (線形結合)×(線形結合)、右辺 = 線形結合** の形。行列で見ると**階数 1 の双線形形式**。単純な乗算ゲート1つを表す。

### 1.3 一般の制約

複雑な式も、「乗算の連鎖」に分解すれば R1CS に入る。たとえば:

$$
x^3 + x^2 = y
$$

を R1CS にするには、中間変数 $t_1 = x^2, t_2 = x^3$ を導入して:

$$
t_1 = x \cdot x, \quad t_2 = t_1 \cdot x, \quad t_2 + t_1 = y
$$

各式を R1CS に:

- $t_1 = x \cdot x$: $(x) \cdot (x) = (t_1)$
- $t_2 = t_1 \cdot x$: $(t_1) \cdot (x) = (t_2)$
- $t_2 + t_1 = y$: $(t_2 + t_1) \cdot (1) = (y)$（**線形式を1で掛けるのがトリック**）

### 1.4 なぜ R1CS 形にこだわるのか

R1CS に統一することで:

- SNARK プロトコル（Groth16 など）が1つの形式だけをサポートすればよい
- 回路言語（Circom）やライブラリ（Arkworks）の出力が共通化
- 後の QAP 変換（Article 12）が機械的に可能

## 2. 回路 → R1CS 変換の手順

### 2.1 Step 1: 中間変数の導入

回路の各乗算ゲートの出力に**新しい変数**を割り当てる。加算ゲートは新変数を導入しないこともある（線形式として処理）。

### 2.2 Step 2: 各乗算について R1CS 制約を作成

$z = x \cdot y$ なら:

$$
(x) \cdot (y) = (z)
$$

$\vec{a}_i, \vec{b}_i, \vec{c}_i$ はそれぞれ「$x$ の係数が1、他0」「$y$ の係数が1」「$z$ の係数が1」のベクトル。

### 2.3 Step 3: 出力や等式を制約化

最終出力 $y$ が正しい場合:

$$
(y) \cdot (1) = (y_{\text{expected}})
$$

### 2.4 例: $f(x) = x^3 + x + 5$

回路:
- $t_1 = x \cdot x$
- $t_2 = t_1 \cdot x$
- $y = t_2 + x + 5$

Witness ベクトルを $\vec{w} = (1, x, y, t_1, t_2)^T$ と定義（公開入出力 + 中間変数）。

制約:

$$
\begin{aligned}
(x) \cdot (x) &= (t_1) \\
(t_1) \cdot (x) &= (t_2) \\
(t_2 + x + 5) \cdot (1) &= (y)
\end{aligned}
$$

### 2.5 第1制約の行列表現

$\vec{a}_1, \vec{b}_1, \vec{c}_1$ をそれぞれ $\vec{w}$ の位置に合わせて書くと、$\vec{w} = (1, x, y, t_1, t_2)$:

$$
\vec{a}_1 = (0, 1, 0, 0, 0), \quad \vec{b}_1 = (0, 1, 0, 0, 0), \quad \vec{c}_1 = (0, 0, 0, 1, 0)
$$

検証: $\vec{a}_1 \cdot \vec{w} = x$、$\vec{b}_1 \cdot \vec{w} = x$、$\vec{c}_1 \cdot \vec{w} = t_1$。$x \cdot x = t_1$ が制約。◯

### 2.6 第3制約 (定数5を含む)

$\vec{a}_3 = (5, 1, 0, 0, 1), \quad \vec{b}_3 = (1, 0, 0, 0, 0), \quad \vec{c}_3 = (0, 0, 1, 0, 0)$

検証: 
- $\vec{a}_3 \cdot \vec{w} = 5 \cdot 1 + 1 \cdot x + 1 \cdot t_2 = 5 + x + t_2$
- $\vec{b}_3 \cdot \vec{w} = 1$
- $\vec{c}_3 \cdot \vec{w} = y$

制約: $(5 + x + t_2) \cdot 1 = y$。◯

## 3. Witness ベクトルの構造

### 3.1 Witness の構成要素

Witness $\vec{w}$ は以下を並べたベクトル:

$$
\vec{w} = \big( \underbrace{1}_{\text{定数1}}, \underbrace{x_1, x_2, \ldots, x_{k}}_{\text{公開入力}}, \underbrace{y_1, \ldots, y_{l}}_{\text{出力}}, \underbrace{t_1, t_2, \ldots}_{\text{中間変数}} \big)^T
$$

### 3.2 定数1を含める理由

「線形定数項」を表現するため。$\vec{a} = (5, 0, \ldots)$ なら線形結合の定数部分が 5。これがないと定数項を表せない。

### 3.3 公開入力と Witness

SNARK では:

- **Public input**: Verifier も知っている値（$x, y$ など）
- **Private witness**: Prover だけが知る値（パスワード、プライベートな中間計算）

両方とも $\vec{w}$ に入るが、位置によって区別する。Verifier は $\vec{w}$ の公開部分だけを確認。

### 3.4 Valid Witness の条件

$\vec{w}$ がすべての $m$ 制約を満たすとき、**valid witness** と呼ぶ。

$$
\forall i: (\vec{a}_i \cdot \vec{w})(\vec{b}_i \cdot \vec{w}) = (\vec{c}_i \cdot \vec{w})
$$

SNARK の目標: **Prover が valid witness を知っている** ことを ZK で証明する。

## 4. 行列表現

### 4.1 行列形式

$m$ 制約をまとめて3つの行列で書く:

$$
A \in \mathbb{F}^{m \times (n+1)}, \quad B \in \mathbb{F}^{m \times (n+1)}, \quad C \in \mathbb{F}^{m \times (n+1)}
$$

行 $i$ が $\vec{a}_i, \vec{b}_i, \vec{c}_i$ に対応。

制約は:

$$
(A \vec{w}) \odot (B \vec{w}) = C \vec{w}
$$

ここで $\odot$ は**要素ごとの積 (Hadamard product)**。

### 4.2 行列サイズ

- $m$: 制約数（≈ 乗算ゲート数）
- $n$: 変数数
- 行列は通常**疎 (sparse)**（ほとんどの要素が 0）

### 4.3 実例: $x^3 + x + 5 = y$

Witness: $\vec{w} = (1, x, y, t_1, t_2)^T$

行列:

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

各行が1制約に対応。$A \vec{w} = (x, t_1, 5+x+t_2)^T$、$B \vec{w} = (x, x, 1)^T$、$C \vec{w} = (t_1, t_2, y)^T$。

Hadamard 積:

$$
(A\vec{w}) \odot (B\vec{w}) = (x \cdot x, t_1 \cdot x, (5+x+t_2) \cdot 1)^T = (t_1, t_2, y)^T = C\vec{w}
$$

◯

## 5. 具体例の完全展開

### 5.1 例: 「秘密 $x$ の二乗根 $s$ を知っている」を示す回路

$$
x = s \cdot s, \quad x \text{ は公開}
$$

**変数**:
- 公開入力: $x$
- プライベート witness: $s$

Witness ベクトル: $\vec{w} = (1, x, s)^T$（中間変数不要）

**制約**: $(s) \cdot (s) = (x)$

$$
\vec{a}_1 = (0, 0, 1), \quad \vec{b}_1 = (0, 0, 1), \quad \vec{c}_1 = (0, 1, 0)
$$

### 5.2 例: 「秘密 $s$ を知っていて、$H(s) = h$」（$H$ は単純ハッシュ $s^5 + 1$）

Witness: $\vec{w} = (1, h, s, t_1, t_2, t_3)^T$

中間:
- $t_1 = s \cdot s$ (= $s^2$)
- $t_2 = t_1 \cdot t_1$ (= $s^4$)
- $t_3 = t_2 \cdot s$ (= $s^5$)
- $h = t_3 + 1$

制約:

| $i$ | $\vec{a}_i$ | $\vec{b}_i$ | $\vec{c}_i$ | 意味 |
|---|---|---|---|---|
| 1 | $(0,0,1,0,0,0)$ | $(0,0,1,0,0,0)$ | $(0,0,0,1,0,0)$ | $s \cdot s = t_1$ |
| 2 | $(0,0,0,1,0,0)$ | $(0,0,0,1,0,0)$ | $(0,0,0,0,1,0)$ | $t_1 \cdot t_1 = t_2$ |
| 3 | $(0,0,0,0,1,0)$ | $(0,0,1,0,0,0)$ | $(0,0,0,0,0,1)$ | $t_2 \cdot s = t_3$ |
| 4 | $(1,0,0,0,0,1)$ | $(1,0,0,0,0,0)$ | $(0,1,0,0,0,0)$ | $(1 + t_3) \cdot 1 = h$ |

### 5.3 制約数とコスト

R1CS のコストは主に制約数で決まる:

- Prover 時間: $O(m \log m)$（FFT）
- 証明サイズ: $O(1)$（Groth16 なら 3 点）
- 検証時間: $O(|\text{public input}|)$

大規模回路（$m = 10^7$）でも Prover は数分〜数十分で証明生成可能。

## 6. R1CS の拡張・限界

### 6.1 R1CS の制限

R1CS は「1 乗算 = 1 制約」という設計。これには制限がある:

- **複雑なゲートが書けない**: たとえば「5 入力の多項式ゲート」を1制約で書けない
- **lookup がない**: 「$x$ が特定のテーブルに入っている」を低コストで示せない

### 6.2 Plonkish (PLONK 系) への進化

PLONK 以降の SNARK は **Plonkish** と呼ばれる拡張制約システムを使う:

- **Custom gates**: 複数変数の多項式制約を1ゲートで
- **Lookup argument**: テーブル参照を低コストで
- **Permutation**: ワイヤの同一性を明示的に扱う

R1CS は「単純だが制約が多い」、Plonkish は「複雑だが制約が少ない」。Halo2, Plonky2, Scroll などが採用。

### 6.3 AIR (Algebraic Intermediate Representation)

STARK 系では **AIR** を使う。これは「**実行トレースの隣接行の関係を多項式で表現**」するもの。

- 1列 $= $ 1 レジスタ
- 1行 $= $ 1 ステップ
- 隣接行の制約 $= $ 状態遷移の正しさ

AIR は CPU のような反復計算に向いており、zkVM や zkEVM で主流。

### 6.4 R1CS と AIR/Plonkish の違い

| 形式 | 発想 | 代表 |
|---|---|---|
| **R1CS** | 乗算ゲート単位 | Groth16, libsnark |
| **Plonkish** | カスタムゲート + lookup | PLONK, Halo2, Plonky2 |
| **AIR** | トレース行の遷移 | STARK, Cairo |

### 6.5 R1CS のツール

R1CS を書くためのツール:

- **Circom**: R1CS 専用言語。HDL 的な記述
- **Arkworks**: Rust ライブラリ、R1CS を直接構築
- **ZoKrates**: 高級言語から R1CS コンパイル
- **snarkjs**: Circom の出力を JavaScript で証明生成

## 7. Q&A

### Q1: なぜ「Rank-1」と呼ぶ？

各制約 $(\vec{a} \cdot \vec{w})(\vec{b} \cdot \vec{w})$ は、行列 $\vec{a}\vec{b}^T$ で表される双線形形式だが、この行列のランクが 1。したがって「Rank-1 Constraint」。

### Q2: 線形制約 $3x + 2y = z$ はどう書く？

**トリック**: $(3x + 2y) \cdot 1 = z$ と書く。$\vec{b}$ は $(1, 0, \ldots, 0)$（定数 1 の位置に 1、他は 0）、$\vec{a}$ は $(0, 3, 0, 2, \ldots)$。

### Q3: 1 つの witness が複数の R1CS を同時に満たすのは可能？

**そう設計する**。同じ $\vec{w}$ がすべての制約 $\forall i$ を満たすことが valid witness の条件。

### Q4: R1CS のサイズは制約数に比例？

**制約数 $m$ と変数数 $n$ で決まる**。行列 $A, B, C$ は $m \times (n+1)$ だが疎なので、メモリは $O(\text{non-zero entries})$。典型的にはトータル non-zero が $3m$（各制約に3変数）なので、$O(m)$。

### Q5: R1CS → 回路の逆変換は？

**可能**。各 R1CS 制約は 1 乗算ゲートとその周りの加算に戻せる。ただし回路の「最適化された形」は失われる。

### Q6: 現代の SNARK で R1CS はもう使われない？

**依然重要**。Groth16 はまだ現役（Zcash 旧実装、Tornado Cash で稼働中）。Nova (folding) も R1CS を基礎とする。ただし新規開発では Plonkish や AIR の方が人気。

## 8. まとめ

### 本記事の要点

1. **R1CS** は「$(\vec{a}_i \cdot \vec{w})(\vec{b}_i \cdot \vec{w}) = (\vec{c}_i \cdot \vec{w})$」という制約の集合
2. **Witness** $\vec{w}$ は $(1, \text{公開入力}, \text{出力}, \text{中間変数})$ を並べたベクトル
3. **行列形式**: $(A\vec{w}) \odot (B\vec{w}) = C\vec{w}$（Hadamard 積）
4. **回路 → R1CS 変換**: 乗算ごとに制約を1つ、線形式は $(\vec{a} \cdot \vec{w}) \cdot 1 = (\vec{c} \cdot \vec{w})$
5. **「Rank-1」の由来**: 各制約の双線形形式がランク 1 行列
6. **限界**: カスタムゲート・lookup が書きづらい → Plonkish, AIR が次世代
7. **ツール**: Circom, Arkworks, ZoKrates

### 次の記事（Article 12）へ

次の記事では、R1CS を **多項式** の世界に持ち上げる **QAP (Quadratic Arithmetic Program)** を扱う。ラグランジュ補間で制約行列を多項式に変換し、$P(X) H(X) = Z(X) ・ \ldots$ という多項式等式に書き換える。この変換が Groth16 の心臓部。

### 3行サマリ

- **R1CS = 乗算制約の標準形式**、1 制約 = 1 Rank-1 行列
- **witness $\vec{w}$** が全制約を満たすことを SNARK で ZK 証明
- **Plonkish / AIR** は R1CS の拡張で、より表現力が高い

---

## 参考文献

- Vitalik Buterin. *Quadratic Arithmetic Programs: from Zero to Hero.* Medium, 2016.
- Ariel Gabizon. *PLONK by Hand.* 2019.
- Howard Wu et al. *libsnark.* GitHub, 2017.
- Iden3. *Circom 2 Language Reference.* 2024.
- ZKP MOOC Lecture 3 (UC Berkeley, 2023).
