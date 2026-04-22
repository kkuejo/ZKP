---
title: "[ゼロ知識証明入門シリーズ 20/30] Proof Gadgets 総まとめ — SNARK の部品を一覧化する"
emoji: "🧩"
type: "tech"
topics:
  - zkp
  - snark
  - proofgadget
  - zerotest
  - permutation
published: false
---

**日付**: 2026年4月22日
**学習内容**: 現代 SNARK の多くは、いくつかの**「部品 (proof gadget)」** の組み合わせでできている。PLONK も Halo2 も、その中身は **ZeroTest, SumCheck, ProductCheck, Permutation Check, Prescribed Permutation Check, Lookup, Range Check** などの組み合わせ。本記事ではこれらを一覧化し、**(1) 各 gadget の目的と数式**、**(2) Grand Product による実装**、**(3) gadget の組み合わせで複雑プロトコルを作る**、**(4) PLONK と Halo2 が使う gadget マップ** を整理する。ここまでの記事の総復習も兼ねる。

## 0. 本記事の位置づけ

この記事は「**SNARK を読むための辞書**」。論文や実装で以下のキーワードに出会ったとき、何を指すかすぐ思い出せるようにする。

構成:

- **第1章**: Proof Gadget の枠組み
- **第2章**: ZeroTest
- **第3章**: SumCheck
- **第4章**: ProductCheck
- **第5章**: Permutation Check
- **第6章**: Prescribed Permutation Check
- **第7章**: Lookup（復習）
- **第8章**: Range Check
- **第9章**: Gadget を組み合わせた PLONK / Halo2
- **第10章**: Q&A とまとめ

## 1. Proof Gadget の枠組み

### 1.1 Gadget とは

**Proof Gadget** は、「ある性質を多項式的に検証するための小さなプロトコル」。たとえば:

- ZeroTest: $f \equiv 0$ on $H$
- SumCheck: $\sum_{x \in H} f(x) = 0$
- ProductCheck: $\prod_{x \in H} f(x) = 1$

これらを**直接 SNARK の構成要素**として使う。

### 1.2 Gadget の入力

典型的な gadget は:

- 多項式コミットメント $C_f$
- 評価領域 $H$ (通常単位根集合)
- チャレンジ/評価点

### 1.3 なぜ gadget 化？

- プロトコルを**モジュラー**に設計できる
- 新しいプロトコルは既存 gadget の組み合わせで記述可能
- セキュリティ証明が**局所化**される

## 2. ZeroTest

### 2.1 問題

多項式 $f(X)$ が評価領域 $H$ 上でゼロ: $\forall h \in H, f(h) = 0$ を示す。

### 2.2 数式

$H = \{\omega^0, \omega^1, \ldots, \omega^{n-1}\}$ のとき、消滅多項式:

$$
Z_H(X) = \prod_{i=0}^{n-1} (X - \omega^i) = X^n - 1
$$

主張:

$$
f(X) = Z_H(X) \cdot q(X) \text{ for some } q
$$

### 2.3 プロトコル

1. Prover が $q(X)$ を計算、コミット $C_q$ を送る
2. Verifier がランダム点 $\zeta$ を送る
3. Prover が $f(\zeta), q(\zeta)$ を開封
4. Verifier が $f(\zeta) = Z_H(\zeta) q(\zeta)$ を確認

### 2.4 使われる場面

- PLONK の gate constraint 全体（$H$ 上でゼロ）
- STARK の AIR 制約

## 3. SumCheck（復習）

### 3.1 問題

$$
H = \sum_{x \in \{0,1\}^n} g(x_1, \ldots, x_n)
$$

を示す。

### 3.2 プロトコル

Article 14 参照。$n$ ラウンドで各変数を固定。

### 3.3 用途

- GKR, Spartan, HyperPlonk
- Multilinear Extension を扱う場合の標準

## 4. ProductCheck

### 4.1 問題

$$
\prod_{x \in H} f(x) = 1
$$

を示す。

### 4.2 Grand Product 多項式

$T(X)$ を次で定義:

$$
T(\omega^0) = f(\omega^0), \quad T(\omega^{i+1}) = T(\omega^i) \cdot f(\omega^{i+1})
$$

**終端条件**: $T(\omega^n) = \prod_{i=0}^{n} f(\omega^i)$

上の等式が成立 $\iff T(\omega^{n-1}) = 1$ (積が 1 なら)。

### 4.3 多項式制約

再帰式を多項式等式に:

$$
(T(X) - 1) \cdot L_0(X) = 0 \quad \text{(初期値)}
$$

$$
T(X \cdot \omega) - T(X) \cdot f(X \cdot \omega) \equiv 0 \text{ on } H \quad \text{(更新)}
$$

$$
(T(X) - 1) \cdot L_{n-1}(X) = 0 \quad \text{(終端値)}
$$

### 4.4 用途

- 置換引数の核（PLONK）
- Lookup argument (Plookup)

## 5. Permutation Check

### 5.1 問題

2 つの多項式（または値の列）$f, g$ が**置換関係**: $f$ の値のマルチセットと $g$ のマルチセットが一致。

### 5.2 Trick: Grand Product で

ランダムチャレンジ $\alpha$ を引いて:

$$
\prod_{x \in H} \frac{\alpha - f(x)}{\alpha - g(x)} \stackrel{?}{=} 1
$$

これが成立 $\iff$ $\{f(x)\}_{x \in H}$ と $\{g(x)\}_{x \in H}$ が等しいマルチセット。

**証明**: 左辺 = $\prod(\alpha - f(x)) / \prod(\alpha - g(x))$。両積は変数 $\alpha$ の多項式。それらが全 $\alpha$ で等しい $\iff$ 根マルチセットが同じ $\iff$ 値マルチセットが同じ。

### 5.3 用途

- PLONK の wiring constraint（実は置換引数）
- 一般に「対応の取れたシャッフル」の検証

## 6. Prescribed Permutation Check

### 6.1 問題

Permutation Check は「何らかの置換が存在する」までしか言えない。**特定の置換 $\sigma$ が成立する**ことを示したい。

### 6.2 Plookup/PLONK での解

ポジション識別子 $\text{id}(x) = x$（恒等）と $\sigma(x)$ を使って:

$$
\prod_{x \in H} \frac{f(x) + \beta \cdot \text{id}(x) + \gamma}{f(x) + \beta \cdot \sigma(x) + \gamma} = 1
$$

ここで $\beta, \gamma$ はランダムチャレンジ。

### 6.3 なぜ成立するか

$\sigma$ が恒等置換なら分子分母同じ。一般の $\sigma$ でも、「$i \to \sigma(i)$ のマッピングを識別子にバインドする」働きで、**正しい $\sigma$ のみ**を通す。

### 6.4 用途

- PLONK の wiring constraint（Article 16）
- 配線の一貫性チェック

## 7. Lookup（復習）

### 7.1 問題

witness 値 $\{f_i\}$ すべてがテーブル $\{t_j\}$ に入る: $f_i \in T$。

### 7.2 Plookup の形

Article 18 参照。Sort trick + grand product。

### 7.3 Log-derivative 形式

$$
\sum_i \frac{1}{X - f_i} = \sum_j \frac{m_j}{X - t_j}
$$

### 7.4 用途

- Range check
- XOR, AND, Poseidon S-box

## 8. Range Check

### 8.1 問題

$x \in [0, 2^k)$ を示す。

### 8.2 Lookup 版

テーブル $T = \{0, 1, \ldots, 2^k - 1\}$ で lookup。

### 8.3 Bit-Decomposition 版

$$
x = \sum_{i=0}^{k-1} b_i \cdot 2^i, \quad b_i \in \{0, 1\}
$$

各 $b_i (b_i - 1) = 0$ を追加。

### 8.4 比較

| 方法 | 制約数 | テーブルサイズ |
|---|---|---|
| Bit decomposition | $k + 1$ | - |
| Lookup | 1 | $2^k$ |

$k = 8$ なら Bit decomp が有利。$k = 64$ なら Lookup が有利（テーブル分割で）。

## 9. Gadget を組み合わせた PLONK / Halo2

### 9.1 PLONK の構成

| Gadget | 用途 |
|---|---|
| ZeroTest | Gate constraint（$F_{\text{gate}}(X) = Z_H(X) \cdot T(X)$） |
| Prescribed Permutation Check | Wiring constraint |
| Plookup (任意) | 高度な制約 |

### 9.2 Halo2 の構成

| Gadget | 用途 |
|---|---|
| ZeroTest | Gate constraint |
| Permutation Check | Copy constraint |
| Lookup | テーブル参照 |
| Custom gate (高次) | 複雑計算の 1 ゲート化 |

### 9.3 STARK / Plonky2 の構成

| Gadget | 用途 |
|---|---|
| ZeroTest | AIR 制約 |
| FRI (Low-Degree Test) | 多項式の次数検証 |
| Permutation | 内部一貫性 |
| Lookup | 効率化 |

### 9.4 HyperPlonk の構成

| Gadget | 用途 |
|---|---|
| Sumcheck | AIR 制約の総和 |
| Multilinear Lookup | Log-derivative ベース |
| MLE 評価 | witness の多項式表現 |

### 9.5 Gadget の選択

各 SNARK 設計は「どの gadget を使うか」を最初に決める。証明サイズ、Prover 時間、Setup の性質がそこで決まる。

## 10. Q&A

### Q1: ZeroTest と SumCheck の違いは？

- **ZeroTest**: 値がすべて 0 かどうか
- **SumCheck**: 総和が特定の値かどうか

ZeroTest $\to$ SumCheck (「すべて 0 なら総和も 0」) だが、ZeroTest の方が強い（個々がゼロを要求）。

### Q2: Permutation Check と Prescribed の違いは？

- Permutation: マルチセット一致（置換の存在）
- Prescribed: 特定の置換の成立

後者の方が情報量が多い。

### Q3: Grand Product を使う gadget が多いのはなぜ？

- 積に対する等式は、**積の双方向性**（交換法則）で扱いやすい
- ランダム線形結合との相性が良い

### Q4: Gadget の健全性はどう積み重なる？

**Union bound**: $\varepsilon_1, \varepsilon_2, \ldots$ の gadget を組み合わせると、総健全性誤差は $\leq \sum \varepsilon_i$（最悪ケース）。

### Q5: 新しい gadget を設計するには？

- 何を検証したいかを明確化
- 既存 gadget で書けるか検討
- 書けないなら新しい多項式等式を発明
- 健全性を Schwartz-Zippel などで証明

### Q6: Gadget の実装ライブラリは？

- **Halo2** (Rust): モジュラーなゲート設計
- **Plonky2** (Rust): STARK 用 gadget
- **gnark** (Go): R1CS 系 gadget
- **circomlib**: Circom 用の標準ライブラリ

## 11. まとめ

### 本記事の要点

1. **Proof Gadget** は SNARK の部品、モジュラーに組み合わせられる
2. **ZeroTest**: $f \equiv 0$ on $H$、$f = Z_H \cdot q$ で示す
3. **SumCheck**: 総和の対話的検証
4. **ProductCheck**: 積の対話的検証、Grand Product
5. **Permutation Check**: マルチセット一致
6. **Prescribed Permutation Check**: 特定の $\sigma$
7. **Lookup**: テーブル参照
8. **Range Check**: Lookup or Bit decomposition
9. **PLONK, Halo2, STARK** はこれらの組み合わせで構成

### 次の記事（Article 21）へ

次の記事から **第5部: STARK系**。STARK (Scalable Transparent ARgument of Knowledge) を扱う。SNARK との違いは「**Hash-based で Trusted Setup 不要、量子耐性あり**」。そのための道具として Reed-Solomon 符号と FRI（Fast Reed-Solomon IOP of Proximity）を学ぶ。

### 3行サマリ

- **Proof Gadget = SNARK の再利用可能な部品**（ZeroTest, SumCheck, Permutation, Lookup）
- **Grand Product がしばしば共通の道具**
- **PLONK / Halo2 / STARK は gadget の組み合わせで構成される**

---

## 参考文献

- Ariel Gabizon et al. *PLONK.* ePrint 2019/953.
- Justin Thaler. *Proofs, Arguments, and Zero-Knowledge.* Chapter 7.
- Electric Coin Company. *Halo2 Book.* 2023.
- Alessandro Chiesa et al. *Fractal: Post-Quantum and Transparent Recursive Proofs.* EUROCRYPT 2020.
- ZKP MOOC Lectures 4, 5 (UC Berkeley, 2023).
