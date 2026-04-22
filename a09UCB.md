# ゼロ知識証明 第9回講義：Linear PCPに基づくSNARKs 完全解説

> **対象読者：** 数学・暗号理論の専門知識がない方でも理解できるよう、基礎から丁寧に説明します。

---

## 目次

1. [前提知識：SNARKsとは何か](#1-前提知識snarksとは何か)
2. [この講義のテーマ：Linear PCPに基づくSNARKs](#2-この講義のテーマlinear-pcpに基づくsnarks)
3. [歴史的背景](#3-歴史的背景)
4. [SNARKs構築のパラダイム（考え方の枠組み）](#4-snarks構築のパラダイム考え方の枠組み)
5. [第1部：二次算術プログラム（QAP）](#5-第1部二次算術プログラムqap)
   - 5.1 [回路充足可能性問題とは](#51-回路充足可能性問題とは)
   - 5.2 [回路のトレース（実行記録）](#52-回路のトレース実行記録)
   - 5.3 [セレクタ多項式](#53-セレクタ多項式)
   - 5.4 [マスター多項式と消失多項式](#54-マスター多項式と消失多項式)
   - 5.5 [QAPの定義](#55-qapの定義)
6. [第2部：QAPからSNARKへ](#6-第2部qapからsnarkへ)
   - 6.1 [Linear PCPとは](#61-linear-pcpとは)
   - 6.2 [QAPとLinear PCPの対応](#62-qapとlinear-pcpの対応)
   - 6.3 [双線形ペアリング（Bilinear Pairing）の復習](#63-双線形ペアリングbilinear-pairingの復習)
   - 6.4 [鍵生成（Key Generation）](#64-鍵生成key-generation)
   - 6.5 [証明生成（Prove）](#65-証明生成prove)
   - 6.6 [検証（Verify）](#66-検証verify)
   - 6.7 [課題1：証明が正しい形か確認する方法（KoE / GGM）](#67-課題1証明が正しい形か確認する方法koe--ggm)
   - 6.8 [課題2：同じ証人ベクトルが使われているか確認する方法](#68-課題2同じ証人ベクトルが使われているか確認する方法)
   - 6.9 [課題3：公開入出力の扱い](#69-課題3公開入出力の扱い)
   - 6.10 [プロトコル全体のまとめ（PGHR13）](#610-プロトコル全体のまとめpghr13)
   - 6.11 [PGHR13の性能特性](#611-pghr13の性能特性)
7. [第3部：その他の変種](#7-第3部その他の変種)
   - 7.1 [Rank-1 Constraint System（R1CS）](#71-rank-1-constraint-systemr1cs)
   - 7.2 [Groth16：最小サイズの証明](#72-groth16最小サイズの証明)
   - 7.3 [ゼロ知識性の達成](#73-ゼロ知識性の達成)
8. [全体まとめ](#8-全体まとめ)

---

## 1. 前提知識：SNARKsとは何か

### SNARKsの意味

**SNARK** とは **Succinct Non-interactive ARgument of Knowledge** の略で、日本語にすると「**簡潔な非対話型知識証明**」です。

- **簡潔（Succinct）**：証明が非常に短い（元のデータに比べてずっと小さい）
- **非対話型（Non-interactive）**：証明者と検証者がやり取りせず、証明者が一方的に証明を送るだけでよい
- **知識証明（Argument of Knowledge）**：ある秘密（witness）を「知っている」ことを、秘密自体は明かさずに証明できる

### 直感的な例え

たとえば「私はこの数独パズルの答えを知っている」と証明したいとします。

- **普通の証明**：答えを全部見せる → 答えがバレてしまう
- **SNARKによる証明**：「確かに答えを知っている」という数百バイトの証明だけを送る → 答えはバレない、しかも検証は一瞬

### これまで学んだSNARKs

この講義シリーズでは、様々なSNARKsを学んできました：

| 暗号的基盤 | Trusted Setup | 代表的スキーム |
|---|---|---|
| ペアリング（KZG） | 必要 | Plonk、Interactive Proofs |
| 離散対数 | 不要 | Bulletproofs、Hyrax、Halo2 |
| ハッシュ関数 | 不要 | Stark、Aurora、Brakedown |

今回学ぶ**Linear PCPに基づくSNARKs**は、最も古くから実装されてきたSNARKsで、**最短の証明サイズ（Groth16では3要素）** と**高速な検証**が特長です。

---

## 2. この講義のテーマ：Linear PCPに基づくSNARKs

### 特徴（長所と短所）

**長所：**
- ✅ 証明サイズが最小（Groth16では3つの群要素、約144バイト）
- ✅ 検証が高速（双線形ペアリング演算のみ）

**短所：**
- ❌ 証明者のコストが高い（FFT演算と群指数演算が必要）
- ❌ 回路ごとにTrusted Setupが必要（後述）

---

## 3. 歴史的背景

SNARKsの発展の歴史は次のようになっています：

```
Kilian'92 → Micali'00 → Groth'10 → GGPR'13 (QSP/QAP) →
→ PGHR13, Groth16, ...（実際に実装・使用されているSNARKs）
```

- **Kilian'92, Micali'00**：理論的なPCP（Probabilistically Checkable Proof）に基づく証明
- **IKO'07, Lipmaa'12**：Linear PCPの理論
- **GGPR'13**：QAP（Quadratic Arithmetic Program）という表現を導入
- **PGHR13, Groth16**：実際に使われるSNARKs（Zcashなどのブロックチェーンで利用）

---

## 4. SNARKs構築のパラダイム（考え方の枠組み）

この講義で学ぶSNARKsは、2つの道具を「ブレンド」して作られます：

```
[暗号的道具（ペアリング）] ─┐
                            ├──→ [一般回路のためのSNARK]
[Linear PCP（QAP）] ────────┘
```

- **Linear PCP（QAP）**：「何を証明したいか」を数学的に整理する部分
- **暗号的道具（双線形ペアリング）**：「その証明を安全・簡潔に検証できるようにする」部分

この2つを組み合わせることで、強力なSNARKsが得られます。

---

## 5. 第1部：二次算術プログラム（QAP）

### 5.1 回路充足可能性問題とは

SNARKsで証明したいのは、次のような問題です：

> **回路充足可能性（Circuit Satisfiability）：**  
> 算術回路 $C$ と出力 $y$ が与えられたとき、  
> ある秘密の入力（証人）$w$ が存在して $C(x, w) = y$ となることを証明せよ。

ここで：
- $C$：加算ゲートと乗算ゲートからなる算術回路
- $x$：公開入力（誰でも知っている）
- $w$：秘密入力（証人 = witness、証明者だけが知っている）
- $y$：公開出力

**簡単のため、以降は $x$ が空（公開入力なし）のケースで説明します。**

### 例の回路

以下の例を使って説明します：

```
       ×        +
      / \      / \
     ×   +    ×
    / \ / \  / \
   3  2 1  7 5  4
```

この回路は3つの乗算ゲートを持ちます。実行すると：

- Gate 1（×）: $3 \times 2 = 6$
- Gate 2（×）: $6 \times (1+7) = 6 \times 8 = 48$
- Gate 3（×）: $(1+7) \times (5+4) = 8 \times 9 = 72$

---

### 5.2 回路のトレース（実行記録）

異なるSNARKスキームは、回路の「実行記録」として異なる情報を記録します：

| スキーム | 記録する情報 |
|---|---|
| Interactive Proof | すべてのゲートの値 |
| Plonk | 各ゲートの左入力・右入力・出力 |
| **QAP（今回）** | **各乗算ゲートの入力と出力のみ** |

QAPでは、トレースとして **拡張証人ベクトル** $\mathbf{c}$ を使います：

$$\mathbf{c} = (c_1, c_2, c_3, c_4, c_5, c_6, c_7, c_8, c_9) = (3, 2, 1, 7, 5, 4, 6, 48, 72)$$

これは「回路内のすべての値（入力・中間値・出力）をフラットに並べたもの」です。

乗算ゲートには番号を付けます：
- Gate 1：$3 \times 2 = 6$（$c_1, c_2 \to c_7$）
- Gate 2：$c_7 \times (c_3 + c_4) = 48$（$c_7, c_3+c_4 \to c_8$）
- Gate 3：$(c_3 + c_4) \times (c_5 + c_6) = 72$（$c_3+c_4, c_5+c_6 \to c_9$）

---

### 5.3 セレクタ多項式

QAPの核心は「**セレクタ多項式**」です。これを使って、「どの $c_i$ がどのゲートの左入力・右入力・出力になっているか」を多項式で表現します。

#### 集合 $\Omega$ の定義

乗算ゲートが3つあるので、3点からなる集合を定義します：

$$\Omega = \{\omega, \omega^2, \omega^3\}$$

ここで $\omega$ は体 $\mathbb{F}$ の元（具体的な値は実装依存ですが、ある特定の値）です。

#### 左入力のセレクタ多項式 $l_i(x)$

$l_i(x)$ は「$c_i$ がゲート $j$ の左入力かどうか」を示す多項式です：

- $l_i(\omega^j) = 1$　⟺　$c_i$ はゲート $j$ の左入力
- $l_i(\omega^j) = 0$　⟺　$c_i$ はゲート $j$ の左入力ではない

例の回路では（各 $l_i$ の値を $(\omega, \omega^2, \omega^3)$ の3点で表す）：

$$l_1(x) : (1, 0, 0) \quad \Rightarrow \quad c_1=3 \text{ はGate 1の左入力}$$
$$l_2(x) : (0, 0, 0) \quad \Rightarrow \quad c_2=2 \text{ はどのゲートの左入力でもない}$$
$$l_3(x) : (0, 0, 1) \quad \Rightarrow \quad c_3=1 \text{ はGate 3の左入力}$$
$$l_4(x) : (0, 0, 1) \quad \Rightarrow \quad c_4=7 \text{ はGate 3の左入力}$$
$$l_7(x) : (0, 1, 0) \quad \Rightarrow \quad c_7=6 \text{ はGate 2の左入力}$$

これらは**多項式補間（Lagrange補間）** で求めます。例えば $l_3(x)$ は：

$$l_3(\omega) = 0,\quad l_3(\omega^2) = 0,\quad l_3(\omega^3) = 1$$

を満たす次数2以下の多項式です。

#### 右入力のセレクタ多項式 $r_i(x)$

同様に、「$c_i$ がゲート $j$ の右入力かどうか」を示す多項式：

$$r_2(x) : (1, 0, 0) \quad \Rightarrow \quad c_2=2 \text{ はGate 1の右入力}$$
$$r_3(x) : (0, 1, 0) \quad \Rightarrow \quad c_3=1 \text{ はGate 2の右入力}$$
$$r_4(x) : (0, 1, 0) \quad \Rightarrow \quad c_4=7 \text{ はGate 2の右入力}$$
$$r_5(x) : (0, 0, 1) \quad \Rightarrow \quad c_5=5 \text{ はGate 3の右入力}$$
$$r_6(x) : (0, 0, 1) \quad \Rightarrow \quad c_6=4 \text{ はGate 3の右入力}$$

#### 出力のセレクタ多項式 $o_i(x)$

「$c_i$ がゲート $j$ の出力かどうか」を示す多項式：

$$o_7(x) : (1, 0, 0) \quad \Rightarrow \quad c_7=6 \text{ はGate 1の出力}$$
$$o_8(x) : (0, 1, 0) \quad \Rightarrow \quad c_8=48 \text{ はGate 2の出力}$$
$$o_9(x) : (0, 0, 1) \quad \Rightarrow \quad c_9=72 \text{ はGate 3の出力}$$

---

### 5.4 マスター多項式と消失多項式

#### 集計多項式 $L(x)$, $R(x)$, $O(x)$

証人ベクトル $\mathbf{c}$ とセレクタ多項式を使って、「各ゲートの左入力の合計・右入力の合計・出力の合計」をエンコードする多項式を作ります：

$$L(x) = \sum_{i=1}^{m} c_i \cdot l_i(x)$$

$$R(x) = \sum_{i=1}^{m} c_i \cdot r_i(x)$$

$$O(x) = \sum_{i=1}^{m} c_i \cdot o_i(x)$$

ここで $m = 9$ は $\mathbf{c}$ のサイズです。

**各点での値を確認すると：**

| 点 | $L$ の値 | $R$ の値 | $O$ の値 |
|---|---|---|---|
| $\omega$ | $c_1 = 3$（Gate 1の左入力） | $c_2 = 2$（Gate 1の右入力） | $c_7 = 6$（Gate 1の出力） |
| $\omega^2$ | $c_7 = 6$（Gate 2の左入力） | $c_3 + c_4 = 8$（Gate 2の右入力） | $c_8 = 48$（Gate 2の出力） |
| $\omega^3$ | $c_3 + c_4 = 8$（Gate 3の左入力） | $c_5 + c_6 = 9$（Gate 3の右入力） | $c_9 = 72$（Gate 3の出力） |

#### マスター多項式 $p(x)$

**乗算ゲートの正しさ**とは「左入力 × 右入力 = 出力」ということです。これを一つの多項式で表します：

$$p(x) = L(x) \cdot R(x) - O(x)$$

$$= \left(\sum_{i=1}^{m} c_i \cdot l_i(x)\right) \cdot \left(\sum_{i=1}^{m} c_i \cdot r_i(x)\right) - \sum_{i=1}^{m} c_i \cdot o_i(x)$$

**重要な性質：** 回路が正しく実行されていれば、各ゲートの点で $p$ はゼロになります：

- $p(\omega) = L(\omega) \cdot R(\omega) - O(\omega) = 3 \times 2 - 6 = 0$ ✅
- $p(\omega^2) = 6 \times 8 - 48 = 0$ ✅
- $p(\omega^3) = 8 \times 9 - 72 = 0$ ✅

#### 消失多項式 $V(x)$

集合 $\Omega = \{\omega, \omega^2, \omega^3\}$ 上でゼロになる多項式を**消失多項式（Vanishing Polynomial）** と呼びます：

$$V(x) = (x - \omega)(x - \omega^2)(x - \omega^3)$$

$p(\omega^j) = 0$ が成り立つとき、代数の定理より：

$$p(x) = V(x) \cdot q(x)$$

を満たす多項式 $q(x)$ が存在します。

**つまり「$p$ が $V$ で割り切れる」⟺「すべての乗算ゲートが正しく計算されている」** です。これがQAPの核心です。

---

### 5.5 QAPの定義

Circuit-SATをQAPに変換すると：

> **Circuit-SAT：** ある $w$ が存在して $C(x, w) = y$  
>    ↓ 変換  
> **QAP：** ある $\mathbf{c}$（拡張証人）が存在して $p(x) = V(x) \cdot q(x)$

形式的には：

$$p(x) = \left(\sum_{i=1}^{m} c_i \cdot l_i(x)\right) \cdot \left(\sum_{i=1}^{m} c_i \cdot r_i(x)\right) - \sum_{i=1}^{m} c_i \cdot o_i(x) = V(x) \cdot q(x)$$

- $m$：拡張証人 $\mathbf{c}$ のサイズ（回路内の値の総数）
- $n$：乗算ゲートの数（$= |\Omega|$）

---

## 6. 第2部：QAPからSNARKへ

### 6.1 Linear PCPとは

**PCP（Probabilistically Checkable Proof）** は「証明者がオラクルに証明を書き込み、検証者がそのオラクルにいくつかの点を問い合わせて確認する」という仕組みです。

**Linear PCP**（Ishai-Kushilevitz-Ostrovsky, 2007）は特殊なPCPで：

- 証明者は**ベクトル** $\mathbf{c}$（Linear PCPオラクル）を送る
- 検証者は**内積クエリ**を使って確認する：$\langle \mathbf{c}, \mathbf{q}_1 \rangle = ?$、$\langle \mathbf{c}, \mathbf{q}_2 \rangle = ?$、...

ここで $\langle \mathbf{c}, \mathbf{q} \rangle = \sum_i c_i \cdot q_i$ は内積です。

### 6.2 QAPとLinear PCPの対応

QAPの等式 $p(x) = V(x) \cdot q(x)$ を、ランダムな点 $\gamma$ で評価すると：

$$L(\gamma) \cdot R(\gamma) - O(\gamma) = V(\gamma) \cdot q(\gamma)$$

これを書き換えると：

$$\langle \mathbf{c}, \mathbf{l}(\gamma) \rangle \cdot \langle \mathbf{c}, \mathbf{r}(\gamma) \rangle - \langle \mathbf{c}, \mathbf{o}(\gamma) \rangle = V(\gamma) \cdot \langle \mathbf{q}, \boldsymbol{\gamma} \rangle$$

ここで：
- $\mathbf{l}(\gamma) = (l_1(\gamma), l_2(\gamma), \ldots, l_m(\gamma))$：$\gamma$ でのセレクタ多項式の値ベクトル
- $\mathbf{c}$：拡張証人
- $\mathbf{q}$：$q(x)$ の係数ベクトル

これはまさに **内積クエリ** の形です！証明者は $\mathbf{c}$ と $q(x)$ の係数をオラクルに書き込み、検証者は $\gamma$ を選んで内積で確認します。

しかし問題があります：

1. 検証者が $\gamma$ を先に決めると、不正な証明者が $\gamma$ に合わせて作れてしまう
2. $\gamma$ を秘密にしながら内積クエリを実行したい

これを解決するのが**双線形ペアリング**による暗号化です。

---

### 6.3 双線形ペアリング（Bilinear Pairing）の復習

#### 基本設定

- $\mathbb{G}$：位数 $p$ の（乗法的）巡回群。生成元を $g$
- $\mathbb{G}_T$：ターゲット群
- $e$：ペアリング写像 $e: \mathbb{G} \times \mathbb{G} \to \mathbb{G}_T$

#### 双線形性

$$e(g^x, g^y) = e(g, g)^{xy}$$

つまり：**指数の積**がペアリングによって「見える」ようになります。

#### 重要な性質

$g^x$ と $g^y$ が与えられたとき、$xy$ の値を知らなくても、ある元 $h = g^{xy}$ かどうかをペアリングで確認できます：

$$e(g^x, g^y) = e(h, g) \iff h = g^{xy}$$

これにより「$\tau$ を秘密にしながら $L(\tau) \cdot R(\tau) = O(\tau) + V(\tau) \cdot q(\tau)$ を検証する」ことが可能になります。

---

### 6.4 鍵生成（Key Generation）

Trusted Setupフェーズで、ランダムな秘密値 $\tau$（タウ）を選び、次の鍵を生成します：

**証明鍵（Proving Key, PK）：**

$$g^{l_i(\tau)},\ g^{r_i(\tau)},\ g^{o_i(\tau)} \quad (i = 1, \ldots, m)$$

$$g^{\tau},\ g^{\tau^2},\ \ldots,\ g^{\tau^m}$$

**検証鍵（Verification Key, VK）：**

$$g^{V(\tau)}$$

**極めて重要：** 鍵生成後、$\tau$ は**完全に削除**します。$\tau$ が漏れると偽の証明が作れてしまいます。これが「Trusted Setup」と呼ばれる理由です。

> **なぜ $g^{l_i(\tau)}$ を渡すのか？**  
> $g^{l_i(\tau)}$ を知っていても、$\tau$ の値は離散対数問題の困難性により求められません。しかし、これらの値を使って証明者は $g^{\sum_i c_i \cdot l_i(\tau)}$ を計算できます（多数の $g^{l_i(\tau)}$ の積を $c_i$ 乗することで）。

---

### 6.5 証明生成（Prove）

証明者は秘密の証人 $\mathbf{c}$ を知っているので、次の4つの証明要素を計算します：

$$\pi_1 = g^{\sum_{i=1}^{m} c_i \cdot l_i(\tau)} = g^{L(\tau)}$$

$$\pi_2 = g^{\sum_{i=1}^{m} c_i \cdot r_i(\tau)} = g^{R(\tau)}$$

$$\pi_3 = g^{\sum_{i=1}^{m} c_i \cdot o_i(\tau)} = g^{O(\tau)}$$

$$\pi_4 = g^{q(\tau)}$$

$\pi_4$ の計算には $g^{\tau}, g^{\tau^2}, \ldots, g^{\tau^m}$ を使います（$q(x) = \sum_j q_j x^j$ なので $g^{q(\tau)} = \prod_j (g^{\tau^j})^{q_j}$）。

これら $(\pi_1, \pi_2, \pi_3, \pi_4)$ が**証明 $\pi$** です。

---

### 6.6 検証（Verify）

検証者は受け取った $(\pi_1, \pi_2, \pi_3, \pi_4)$ と検証鍵 $g^{V(\tau)}$ を使って、次の等式をペアリングで確認します：

$$\frac{e(\pi_1, \pi_2)}{e(\pi_3, g)} = e(g^{V(\tau)}, \pi_4)$$

**なぜこれで確認できるか？**

左辺を展開：
$$e(g^{L(\tau)}, g^{R(\tau)}) / e(g^{O(\tau)}, g) = e(g, g)^{L(\tau) \cdot R(\tau)} / e(g,g)^{O(\tau)} = e(g,g)^{L(\tau) \cdot R(\tau) - O(\tau)}$$

右辺を展開：
$$e(g^{V(\tau)}, g^{q(\tau)}) = e(g,g)^{V(\tau) \cdot q(\tau)}$$

等式が成り立つ ⟺ $L(\tau) \cdot R(\tau) - O(\tau) = V(\tau) \cdot q(\tau)$

これはシュワルツ-ジッペル補題により、多項式の等式 $L(x) \cdot R(x) - O(x) = V(x) \cdot q(x)$ が成立する場合に（ほぼ確実に）起こります。

---

### 6.7 課題1：証明が正しい形か確認する方法（KoE / GGM）

**問題：** 不正な証明者が $\pi_1 = g^{L(\tau)}$ ではなく、でたらめな群の元を送っても検証を通り抜けられないか？

**解決策1：Knowledge of Exponent (KoE) 仮定 [PGHR13]**

追加のランダム値 $\alpha$ を秘密に選び、$g^{\alpha \cdot l_i(\tau)}$ も証明鍵に加えます。

証明者は：
- $\pi_1 = g^{L(\tau)}$
- $\pi_1' = g^{\alpha \cdot L(\tau)}$

の両方を送り、検証者は：

$$e(\pi_1, g^{\alpha}) = e(\pi_1', g)$$

を確認します。$\pi_1' = ({\pi_1})^{\alpha}$ が成り立つとき、$\pi_1$ は確かに $g^{l_i(\tau)}$ の線形結合です（KoE仮定のもとで）。

**解決策2：Generic Group Model (GGM) [Groth16]**

「敵対者（攻撃者）はグループ演算のオラクルしか使えない」という仮定のもとでは、$g^{l_i(\tau)}$ が与えられた場合、その**線形結合**しか計算できません。これにより $\pi_1$ が正しい形であることが保証されます。

---

### 6.8 課題2：同じ証人ベクトルが使われているか確認する方法

**問題：** $\pi_1, \pi_2, \pi_3$ に異なる $\mathbf{c}$（$\mathbf{c}_L, \mathbf{c}_R, \mathbf{c}_O$）を使って不正な証明を作れるか？

**解決策：** 追加の秘密値 $\beta$ を使います。

証明鍵に以下を追加：

$$g^{\beta(l_i(\tau) + r_i(\tau) + o_i(\tau))} \quad (i \in [m])$$

$$g^{\beta}$$

証明者は追加で $\pi_5$ を計算して送ります：

$$\pi_5 = \prod_{i=1}^{m} \left(g^{\beta(l_i(\tau) + r_i(\tau) + o_i(\tau))}\right)^{c_i} = g^{\beta(L(\tau) + R(\tau) + O(\tau))}$$

検証者は追加で以下を確認：

$$e(\pi_1 \cdot \pi_2 \cdot \pi_3,\ g^{\beta}) = e(\pi_5, g)$$

もし $\pi_1, \pi_2, \pi_3$ に同じ $\mathbf{c}$ が使われていれば：
$$e(g^{L(\tau)+R(\tau)+O(\tau)}, g^{\beta}) = e(g^{\beta(L(\tau)+R(\tau)+O(\tau))}, g)$$

が成立し、異なる $\mathbf{c}$ を使った場合は（ほぼ確実に）失敗します。

---

### 6.9 課題3：公開入出力の扱い

**問題：** $C(x, w) = y$ のとき、公開入出力 $x, y$ も回路に含まれますが、これらは秘密にすべきではありません。

**解決策：** 証人の添字集合を2つに分けます：

- $I_{mid}$：秘密の証人に対応する添字
- $I_{io}$：公開入出力に対応する添字

証明者は $\pi_1$ として**秘密部分のみ**を送ります：

$$\pi_1 = g^{\sum_{i \in I_{mid}} c_i \cdot l_i(\tau)}$$

検証者は公開入出力 $(x, y)$ を知っているので、自分で公開部分を計算して加えます：

$$\pi_1^* = \pi_1 \cdot g^{\sum_{i \in I_{io}} c_i \cdot l_i(\tau)}$$

こうして、検証者は $\pi_1^*$ を使って同じ検証を行います。

---

### 6.10 プロトコル全体のまとめ（PGHR13）

**鍵生成（KeyGen）：**

- 証明鍵 PK：
  $$g^{l_i(\tau)},\ g^{r_i(\tau)},\ g^{o_i(\tau)},\ g^{\beta(l_i(\tau)+r_i(\tau)+o_i(\tau))} \quad (i \in I_{mid})$$
  $$g^{\beta},\ g^{\tau},\ g^{\tau^2},\ \ldots,\ g^{\tau^m}$$

- 検証鍵 VK：
  $$g^{l_i(\tau)},\ g^{r_i(\tau)},\ g^{o_i(\tau)} \quad (i \in I_{io}),\quad g^{V(\tau)}$$

**証明（Prove）：** $(\pi_1, \pi_2, \pi_3, \pi_4, \pi_5)$ を計算して送る

**検証（Verify）：**

1. $\pi_j^* = \pi_j \cdot g^{\sum_{i \in I_{io}} c_i \cdot \bullet_i(\tau)}$ を計算（$j = 1,2,3$）
2. 乗算ゲートの正しさを確認：
   $$e(\pi_1^*, \pi_2^*) / e(\pi_3^*, g) = e(g^{V(\tau)}, \pi_4)$$
3. 同じ証人が使われているか確認：
   $$e(\pi_1 \cdot \pi_2 \cdot \pi_3,\ g^{\beta}) = e(\pi_5, g)$$

---

### 6.11 PGHR13の性能特性

| 項目 | 複雑度 |
|---|---|
| Trusted Setup | $O(|C|)$ 回の群指数演算（回路サイズに比例） |
| 証明者計算量 | $O(|C| \log |C|)$ のFFT + $O(|C|)$ の群指数演算 |
| **証明サイズ** | $O(1)$（数百バイト！） |
| **検証時間** | $O(1)$ のペアリング + $O(|IO|)$ の群指数演算 |

証明サイズと検証時間が定数（$O(1)$）なのが最大の強みです。

---

## 7. 第3部：その他の変種

### 7.1 Rank-1 Constraint System（R1CS）

QAPをさらに一般化したのが **R1CS** です。

#### QAPからR1CSへの拡張

QAPでは $l_i(\omega^j) \in \{0, 1\}$（0か1のみ）でしたが、R1CSでは**任意の公開定数**を使えます：

$$l_i(\omega^j) \in \mathbb{F} \quad \text{（任意の体の元）}$$

例えば、次のような制約を表現できます：

$$\text{制約}: (3c_1 + 5c_5 - 7c_7) \times (6c_2 + 10c_9 - c_3 - 2c_8) = 0$$

これは以下のように設定：

$$l_1(\omega) = 3,\quad l_5(\omega) = 5,\quad l_7(\omega) = -7$$
$$r_2(\omega) = 6,\quad r_9(\omega) = 10,\quad \ldots$$

#### 行列表現

R1CSを行列で表現すると非常にすっきりします：

$$m: \text{拡張証人のサイズ}, \quad n: \text{制約（乗算ゲート）の数}$$

$$L \cdot \mathbf{c}\ \otimes\ R \cdot \mathbf{c} = O \cdot \mathbf{c}$$

ここで $L, R, O$ は $n \times m$ 行列、$\otimes$ はアダマール積（要素ごとの積）です。

R1CSは現在、多くのSNARKシステム（Bulletproofs、Marlin、Spartan等）の基本となっています。

---

### 7.2 Groth16：最小サイズの証明

**Groth16**（Jens Groth, 2016年）は、QAPを使いつつ証明をさらに最適化した画期的なSNARKです。

#### 証明の構造

3つの群要素だけで証明を構成します：

$$\pi_1 = g^{\alpha + \sum_{i=1}^{m} c_i \cdot l_i(\tau)}$$

$$\pi_2 = g^{\beta + \sum_{i=1}^{m} c_i \cdot r_i(\tau)}$$

$$\pi_3 = g^{\sum_i c_i (\beta \cdot l_i(\tau) + \alpha \cdot r_i(\tau) + o_i(\tau)) + V(\tau) \cdot q(\tau)}$$

#### 検証等式

$$e(\pi_1, \pi_2) = e(\pi_3, g) \cdot e(g^{\alpha}, g^{\beta})$$

ただ1回のペアリング等式で検証できます！

#### 性能

| 項目 | 値 |
|---|---|
| **証明サイズ** | **3つの群要素（144バイト！）** |
| **検証時間** | **1回のペアリング等式** |

Groth16は現在も最小の証明サイズを持つSNARKとして、Zcash（Sapling）などで実際に使われています。

---

### 7.3 ゼロ知識性の達成

今まで説明したプロトコルは「**証明の健全性（Soundness）**」は保証しますが、そのままでは**完全なゼロ知識性**がありません。

**問題：** $\pi_1 = g^{L(\tau)}$ を見れば、$L(\tau)$ の情報が漏れる可能性があります。

**解決策：** ランダムな「マスク値」$\delta_1, \delta_2, \delta_3$ を証明に追加します：

$$\pi_1 = g^{\sum_{i=1}^{m} c_i \cdot l_i(\tau) + \delta_1 \cdot V(\tau)}$$

$$\pi_2 = g^{\sum_{i=1}^{m} c_i \cdot r_i(\tau) + \delta_2 \cdot V(\tau)}$$

$$\pi_3 = g^{\sum_{i=1}^{m} c_i \cdot o_i(\tau) + \delta_3 \cdot V(\tau)}$$

**なぜこれで検証が通るか？**

マスクが追加されても $p(x) = V(x) q(x)$ の形は保たれます（$q(x)$ を適切に修正することで）。そして各 $\pi_j$ はランダムに見えるため、$\mathbf{c}$ の情報は一切漏れません。

これが**ゼロ知識性（Zero-Knowledge）** の達成方法です：証明を見ても秘密の証人 $\mathbf{c}$ について何も学べません。

---

## 8. 全体まとめ

### SNARKs（Linear PCPベース）の完全な構築フロー

```
[算術回路 C] 
    ↓ （Circuit-SAT to QAP変換）
[QAP: p(x) = V(x)q(x)]
    ↓ （Linear PCP化）
[Linear PCPクエリ: <c, l(γ)>・<c, r(γ)> - <c, o(γ)> = V(γ)<q, γ>]
    ↓ （双線形ペアリングによる暗号化）
[SNARK: π = (π₁, π₂, π₃, π₄), 検証は1回のペアリング等式]
```

### 各スキームの比較

| 項目 | PGHR13 | Groth16 |
|---|---|---|
| 証明サイズ | 数百バイト | 144バイト（最小！） |
| 検証等式 | 2回のペアリング | 1回のペアリング |
| 根拠仮定 | KoE | GGM |
| Trusted Setup | 回路ごと | 回路ごと |

### キーワードまとめ

| 用語 | 意味 |
|---|---|
| QAP | 回路の充足可能性を多項式の割り算問題に変換した表現 |
| セレクタ多項式 | 「$c_i$ がゲート $j$ の左/右/出力か」を多項式でエンコード |
| 消失多項式 $V(x)$ | すべての乗算ゲートの点でゼロになる多項式 |
| Linear PCP | 証人ベクトルへの内積クエリによる確率的証明 |
| 双線形ペアリング | 秘密の $\tau$ を隠しながら指数の積を検証できる暗号道具 |
| Trusted Setup | $\tau$ を選んで鍵を作り、その後 $\tau$ を削除するフェーズ |
| R1CS | QAPを一般化した制約表現（現代のSNARKの基本） |
| Groth16 | 3群要素・1ペアリング等式の世界最小証明SNARK |
| ゼロ知識性 | 証明を見ても秘密の証人が一切漏れない性質 |

### 次の講義

この講義の最後で予告されているように、次は**Recursive SNARKs（再帰的SNARK）** を学びます。これはSNARKの証明自体を別のSNARKで証明するという強力な技法で、Proof of ProofやProof Aggregationなどに使われます。

---

*本資料はZKP MOOC Lecture 9「SNARKs based on Linear PCP」の内容を基に、初学者向けに詳細解説したものです。*
*数式はLaTeX形式で記述しています。*
