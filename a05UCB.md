# ゼロ知識証明と PLONK SNARK：完全解説

> **対象読者**: 暗号理論の予備知識がない方でも理解できるように、基礎から丁寧に解説します。

---

## 目次

1. [ゼロ知識証明とは何か？](#1-ゼロ知識証明とは何か)
2. [SNARKとは何か？](#2-snarkとは何か)
3. [多項式コミットメントスキーム（PCS）](#3-多項式コミットメントスキームpcs)
4. [KZG多項式コミットメント](#4-kzg多項式コミットメント)
5. [KZGの評価証明（Eval Proof）](#5-kzgの評価証明eval-proof)
6. [KZGの性質と応用](#6-kzgの性質と応用)
7. [コミット済み多項式の性質証明](#7-コミット済み多項式の性質証明)
8. [重要な証明ガジェット（Proof Gadgets）](#8-重要な証明ガジェットproof-gadgets)
9. [PLONK：汎用回路のためのSNARK](#9-plonk汎用回路のためのsnark)
10. [PLONKの完全プロトコル](#10-plonkの完全プロトコル)
11. [まとめ](#11-まとめ)

---

## 1. ゼロ知識証明とは何か？

### 直感的な説明

「ゼロ知識証明（Zero-Knowledge Proof）」とは、**ある事実を知っていることを、その事実の内容を一切明かさずに証明する**技術です。

**日常的な例え：**
- あなたが「この絵の中に隠れているキャラクターを見つけた」と友人に証明したいとします。
- でも、キャラクターの場所を教えたくない。
- そこで、大きな紙でキャラクターの場所だけ切り抜いて見せれば、「見つけた」ことは証明できます。友人は場所を知りません。

### 暗号技術での役割

ブロックチェーンや暗号通貨では：
- 「私はこのトランザクションの正当な署名者だ」
- 「この計算結果は正しい」
- 「私はこの条件を満たしている（残高 > 0など）」

これらを**秘密情報を漏らさずに**証明したい場面が多くあります。

---

## 2. SNARKとは何か？

### SNARKの定義

**SNARK**（Succinct Non-interactive ARgument of Knowledge）は以下の特徴を持つ証明システムです：

| 特徴 | 意味 |
|------|------|
| **Succinct（簡潔）** | 証明のサイズが非常に小さい（計算の大きさに関係なく） |
| **Non-interactive（非対話型）** | 証明者と検証者がリアルタイムで会話しなくてよい |
| **ARgument of Knowledge** | 証明者が「証人」（秘密の解）を本当に知っていることを保証 |

### SNARKの構築方法

SNARKは2つの部品を組み合わせて作ります：

```
┌─────────────────────────┐
│  多項式コミットメントスキーム │
│    (Polynomial           │──────┐
│  Commitment Scheme)      │      ▼
└─────────────────────────┘   ┌──────────────────┐
                               │  SNARK for       │
┌─────────────────────────┐   │  General Circuits│
│ 多項式対話型オラクル証明  │──▶│  (汎用回路用SNARK) │
│ (Polynomial IOP)        │   └──────────────────┘
└─────────────────────────┘
```

この講義では、**PLONK**という手法でこれを実現する方法を学びます。

---

## 3. 多項式コミットメントスキーム（PCS）

### 多項式とは？

まず「多項式」を思い出しましょう。中学校で習った

$$f(X) = f_0 + f_1 X + f_2 X^2 + \cdots + f_d X^d$$

という形の式です。$f_0, f_1, \ldots, f_d$ は「係数」と呼ばれる数値です。$d$ は「次数（degree）」です。

### コミットメントとは？

「コミットメント」とは、**封筒に入れて封をする**イメージです：
- 封筒の中身（多項式の係数）は隠れている
- でも後で「封筒に入っているのはこの多項式だ」と証明できる
- 一度封をしたら中身を変更できない（**束縛性: binding**）

### PCSの仕組み

**セットアップ（Setup）：**
信頼できる第三者がシステムのパラメータ $gp$（global parameters）を生成します。

**コミット（Commit）：**
証明者（Prover）が多項式 $f(X)$ に対してコミットメント $\text{com}_f$ を計算します。

$$\text{com}_f = \text{commit}(gp, f) \in \mathbb{G}$$

ここで $\mathbb{G}$ は楕円曲線の群（後述）です。

**評価証明（Eval Proof）：**
公開された点 $u, v \in \mathbb{F}_p$ に対して、証明者は

$$f(u) = v \quad \text{かつ} \quad \deg(f) \leq d$$

であることを検証者（Verifier）に証明できます。

**効率性の要件：**
評価証明のサイズと検証者の計算時間は $O_\lambda(\log d)$ であるべきです。

---

## 4. KZG多項式コミットメント

### 楕円曲線の群（Group）

KZGスキームは**楕円曲線暗号**を使います。楕円曲線上の点の集合を

$$\mathbb{G} \coloneqq \{0, G, 2 \cdot G, 3 \cdot G, \ldots, (p-1) \cdot G\}$$

と書きます。ここで：
- $G$ は「生成元（Generator）」と呼ばれる基準点
- $p$ は大きな素数（群の位数）
- $n \cdot G$ は「スカラー倍算」（$G$ を $n$ 回足すこと）

**重要な性質（離散対数問題）：**
$n \cdot G$ から $n$ を逆算することは計算上不可能です。これが安全性の根拠です。

### セットアップ手順

**KZGセットアップ $\text{setup}(1^\lambda) \to gp$：**

1. ランダムな秘密値 $\tau \in \mathbb{F}_p$ を選ぶ
2. グローバルパラメータを計算する：

$$gp = \left(H_0 = G,\ H_1 = \tau \cdot G,\ H_2 = \tau^2 \cdot G,\ \ldots,\ H_d = \tau^d \cdot G\right) \in \mathbb{G}^{d+1}$$

3. **$\tau$ を削除する！**（これが「信頼されたセットアップ」）

> **なぜ $\tau$ を削除するのか？**
> $\tau$ が漏れると、偽の証明を作れてしまいます。セレモニー（複数人で安全に $gp$ を生成し誰も $\tau$ を知らないようにする）と呼ばれる手続きが実際に使われます。

### コミットの計算

多項式 $f(X) = f_0 + f_1 X + \cdots + f_d X^d$ に対して：

$$\text{com}_f \coloneqq f(\tau) \cdot G \in \mathbb{G}$$

実際の計算は：

$$\text{com}_f = f_0 \cdot H_0 + f_1 \cdot H_1 + \cdots + f_d \cdot H_d$$

$$= f_0 \cdot G + f_1 \cdot (\tau G) + f_2 \cdot (\tau^2 G) + \cdots = f(\tau) \cdot G$$

**なぜこれがコミットメントになるのか？**
- $gp$ だけ見ても $\tau$ の値はわからない（離散対数問題）
- よって $f(\tau)$ の値も直接にはわからない
- でも証明者は係数 $f_0, \ldots, f_d$ を知っているので計算できる

**注意：KZGは「束縛性あり・隠蔽性なし」：**
- $\text{com}_f$ から $f$ 自体を復元することは困難（安全）
- ただし、ゼロ知識性（完全な隠蔽性）は持たない（基本形では）

---

## 5. KZGの評価証明（Eval Proof）

### 目標

証明者は「コミットした多項式 $f$ は点 $u$ で値 $v$ を取る」、つまり $f(u) = v$ を証明したい。

### 核心的なアイデア

**代数的な観察：**

$$f(u) = v$$

$$\iff u \text{ は } \hat{f}(X) \coloneqq f(X) - v \text{ の根}$$

$$\iff (X - u) \text{ が } \hat{f}(X) \text{ を割り切る}$$

$$\iff \exists q(X) \in \mathbb{F}_p[X] \text{ s.t. } q(X) \cdot (X - u) = f(X) - v$$

つまり、**商多項式 $q(X)$ が存在することが $f(u) = v$ の証拠**になります。

### プロトコルの流れ

```
証明者 (Prover)                        検証者 (Verifier)
f(X), u, v を知っている                com_f, u, v を知っている

q(X) = (f(X) - v) / (X - u) を計算
com_q = q(τ) · G を計算

      π := com_q ∈ G
      ─────────────────────────────────────▶
                                           (τ - u) · com_q = com_f - v · G
                                           を確認して accept/reject
```

**証明サイズ：** $\pi = \text{com}_q$ は群 $\mathbb{G}$ の1要素（約32バイト）。次数 $d$ に無関係！

### 検証の仕組み

検証者は $\tau$ を知らないのに、どうやって確認するのか？

検証条件を展開すると：

$$(\tau - u) \cdot q(\tau) \cdot G \stackrel{?}{=} (f(\tau) - v) \cdot G$$

これは **ペアリング（Pairing）** という楕円曲線上の特殊な演算を使って確認できます。ペアリングは2つの群の要素を掛け合わせる操作で、$gp$ の $H_0 = G$ と $H_1 = \tau \cdot G$ だけあれば検証できます。

**証明者の計算コスト：** $q(X)$ の計算は次数 $d$ に対して線形時間ですが、大きな $d$ では重い。

---

## 6. KZGの性質と応用

### 線形時間コミット

多項式を2通りの方法で表現できます：

#### 係数表現

$$f(X) = f_0 + f_1 X + \cdots + f_d X^d$$

コミットの計算：

$$\text{com}_f = f_0 \cdot H_0 + \cdots + f_d \cdot H_d$$

→ $d$ に対して**線形時間** $O(d)$

#### 点値表現

$$\{(a_0, f(a_0)),\ (a_1, f(a_1)),\ \ldots,\ (a_d, f(a_d))\}$$

朴素な方法では係数を求めてから計算するため $O(d \log d)$ かかります。

#### ラグランジュ補間を使う改良

**ラグランジュ補間の定理：**

$$f(\tau) = \sum_{i=0}^{d} \lambda_i(\tau) \cdot f(a_i)$$

ここでラグランジュ基底多項式は：

$$\lambda_i(\tau) = \frac{\prod_{j=0, j \neq i}^{d} (\tau - a_j)}{\prod_{j=0, j \neq i}^{d} (a_i - a_j)} \in \mathbb{F}_p$$

**アイデア：** $gp$ をラグランジュ形式に変換する

$$\widehat{gp} = \left(\hat{H}_0 = \lambda_0(\tau) \cdot G,\ \hat{H}_1 = \lambda_1(\tau) \cdot G,\ \ldots,\ \hat{H}_d = \lambda_d(\tau) \cdot G\right)$$

すると：

$$\text{com}_f = f(\tau) \cdot G = f(a_0) \cdot \hat{H}_0 + \cdots + f(a_d) \cdot \hat{H}_d$$

→ 点値から直接 $O(d)$ でコミット計算可能！

### 高速マルチポイント証明生成

証明者が多項式 $f(X)$ と集合 $\Omega \subseteq \mathbb{F}_p$（$|\Omega| = d$）を持つとき、全ての $a \in \Omega$ に対する評価証明を生成する場合：

- **朴素な方法：** $O(d^2)$（各証明に $O(d)$ かかり、$d$ 個）
- **FK（Feist-Khovratovich）アルゴリズム（2020年）：**
  - $\Omega$ が乗法部分群の場合：$O(d \log d)$
  - それ以外：$O(d \log^2 d)$

### Dory：信頼されたセットアップなしのPCS

KZGの問題点：信頼されたセットアップが必要で、$gp$ のサイズが $d$ に比例して大きい。

**Dory スキームの特徴：**

| 特性 | 内容 |
|------|------|
| セットアップ | **透明**（秘密の乱数不要） |
| コミットサイズ | 1つの群要素（次数 $d$ 非依存） |
| 評価証明サイズ | $O(\log d)$ 個の群要素 |
| 検証時間 | $O(\log d)$ |
| 証明者時間 | $O(d)$ |

### PCSの応用：ベクトルコミットメント

KZGはMerkleツリーの代替として使えます。

**Merkleツリーの問題：** 要素数が増えると証明サイズが $O(\log n)$ になる。

**KZGを使ったベクトルコミットメント：**

ベクトル $(u_1, \ldots, u_k)$ に対して：
1. 多項式 $f$ を補間：$f(i) = u_i$ for $i = 1, \ldots, k$
2. $\text{com}_f \coloneqq \text{commit}(gp, f)$ を計算

「$u_2 = a$ かつ $u_4 = b$」の証明：
- $f(2) = a$ かつ $f(4) = b$ の評価証明 $\pi \in \mathbb{G}$ を送る
- **$\pi$ は群の1要素のみ！** Merkleツリーより短い

---

## 7. コミット済み多項式の性質証明

### 多項式IOP（Polynomial Interactive Oracle Proof）

証明者は多項式 $f, g$ を持ち、検証者はそのコミットメント $\boxed{f}, \boxed{g}$ を持ちます。

プロトコルはIOP（対話型オラクル証明）として設計されます：

```
証明者 P(f, g)                         検証者 V(⌈f⌉, ⌈g⌉)

                    ←──────────── r ──────────── r ←$ F_p
      補助多項式 q を計算
                    ──────── ⌈q⌉ ─────────────────▶

                    ←────────────────────────────── f(X), g(X), q(X) を点 r で聞く

      f(r), g(r), q(r) と証明を送る
                    ──────────────────────────────▶ accept/reject
```

### 多項式等値テスト

**Schwartz-Zippelの補題（直感）：**

$p \approx 2^{256}$（超巨大な素数）、$d \leq 2^{40}$ のとき、$d/p$ は無視できるほど小さい。

このとき：

$$r \xleftarrow{\$} \mathbb{F}_p \text{ で } f(r) = g(r) \implies f = g \text{ （高確率）}$$

なぜなら、$f \neq g$ の場合、$f(X) - g(X)$ は高々 $d$ 個の根を持ちます。ランダムな $r$ がその根になる確率は $d/p$ 以下（無視できる）。

これにより**コミットされた多項式の等値テスト**が可能：
$$f(r) - g(r) = 0 \implies f - g = 0 \text{ （高確率）}$$

---

## 8. 重要な証明ガジェット（Proof Gadgets）

$\Omega$ を $\mathbb{F}_p$ の部分集合（サイズ $k$）とします。具体的には：

$$\Omega = \{1, \omega, \omega^2, \ldots, \omega^{k-1}\}$$

ここで $\omega$ は $\mathbb{F}_p$ の $k$ 次の原始根の単位元（$\omega^k = 1$）。

### 消滅多項式（Vanishing Polynomial）

$$Z_\Omega(X) \coloneqq \prod_{a \in \Omega} (X - a)$$

重要な性質：$\Omega = \{1, \omega, \ldots, \omega^{k-1}\}$ のとき

$$Z_\Omega(X) = X^k - 1$$

$r \in \mathbb{F}_p$ での評価は $\leq 2\log_2 k$ 回の演算で計算可能。

---

### ガジェット1：ZeroTest on $\Omega$

**目標：** $f$ が $\Omega$ 上で恒等的に0であることを証明する

**鍵となる補題：**

$$f \text{ が } \Omega \text{ 上でゼロ} \iff Z_\Omega(X) \text{ が } f(X) \text{ を割り切る}$$

$$\iff \exists q(X) \in \mathbb{F}_p[X] \text{ s.t. } f(X) = q(X) \cdot Z_\Omega(X)$$

**プロトコル：**

```
証明者 P(f)                             検証者 V(⌈f⌉)

q(X) ← f(X) / Z_Ω(X) を計算
       ──── ⌈q⌉ ──────────────────────▶
                                          r ←$ F_p
       ←──────── f(X) と q(X) を r で聞く ──
                                          
f(r) と q(r) を送る
       ──────────────────────────────────▶ f(r) =? q(r) · Z_Ω(r) を確認
                                           （V は Z_Ω(r) = r^k - 1 を計算）
```

**定理：** このプロトコルは完全かつ健全（soundness）、$d/p$ が無視できるほど小さければ。

**計算量：**
- 検証者時間：$O(\log k)$
- 証明者時間：$q(X)$ の計算とコミットが支配的

---

### ガジェット2：SumCheck on $\Omega$

**目標：** $\sum_{a \in \Omega} f(a) = 0$ を証明する

これは積確認（ProdCheck）から導出できます。

---

### ガジェット3：ProdCheck on $\Omega$

**目標：** $\prod_{a \in \Omega} f(a) = 1$ を証明する

**鍵となるアイデア：** 補助多項式 $t \in \mathbb{F}_p^{(\leq k)}[X]$ を次のように定義する：

$$t(1) = f(1), \quad t(\omega^s) = \prod_{i=0}^{s} f(\omega^i) \quad \text{for } s = 1, \ldots, k-1$$

すると：
- $t(\omega) = f(1) \cdot f(\omega)$
- $t(\omega^2) = f(1) \cdot f(\omega) \cdot f(\omega^2)$
- $\vdots$
- $t(\omega^{k-1}) = \prod_{a \in \Omega} f(a)$

**補題：** 以下の2条件が成立 $\implies \prod_{a \in \Omega} f(a) = 1$

1. $t(\omega^{k-1}) = 1$（最終積が1）
2. $t(\omega \cdot x) - t(x) \cdot f(\omega \cdot x) = 0 \quad \forall x \in \Omega$（累積積の漸化式）

条件2は $\Omega$ 上での**ZeroTest**で確認できます。

**プロトコル（非最適化版）：**

$$t_1(X) \coloneqq t(\omega \cdot X) - t(X) \cdot f(\omega \cdot X)$$

$$q(X) = t_1(X) / (X^k - 1)$$

検証者は：
- $t(\omega^{k-1}) \stackrel{?}{=} 1$
- $t(\omega r) - t(r) f(\omega r) \stackrel{?}{=} q(r) \cdot (r^k - 1)$

**性能：**
- 証明サイズ：コミット2個 + 評価値5個
- 検証者時間：$O(\log k)$
- 証明者時間：$O(k \log k)$

---

### 有理関数への拡張：$\prod_{a \in \Omega} (f/g)(a) = 1$

同様のアイデアで、有理関数の積確認もできます。

$t$ を次のように定義：

$$t(1) = f(1)/g(1), \quad t(\omega^s) = \prod_{i=0}^{s} \frac{f(\omega^i)}{g(\omega^i)} \quad \text{for } s = 1, \ldots, k-1$$

**補題：** 以下の2条件が成立 $\implies \prod_{a \in \Omega} f(a)/g(a) = 1$

1. $t(\omega^{k-1}) = 1$
2. $\boxed{t(\omega \cdot x) \cdot g(\omega \cdot x) = t(x) \cdot f(\omega \cdot x)} \quad \forall x \in \Omega$

---

### ガジェット4：Permutation Check（置換確認）

**目標：** $f$ の値列が $g$ の値列の置換であることを証明する

$$\left(f(1), f(\omega), f(\omega^2), \ldots, f(\omega^{k-1})\right) \text{ は } \left(g(1), g(\omega), g(\omega^2), \ldots, g(\omega^{k-1})\right) \text{ の並べ替え}$$

**アイデア（Liptonのトリック, 1989）：**

$$\hat{f}(X) \coloneqq \prod_{a \in \Omega} (X - f(a)), \quad \hat{g}(X) \coloneqq \prod_{a \in \Omega} (X - g(a))$$

$$\hat{f}(X) = \hat{g}(X) \iff g \text{ は } f \text{ の置換}$$

**プロトコル：**
1. 検証者がランダム $r \xleftarrow{\$} \mathbb{F}_p$ を選ぶ
2. 証明者が $\hat{f}(r) = \hat{g}(r)$ を積確認（ProdCheck）で証明：

$$\text{ProdCheck}: \quad \prod_{a \in \Omega} \frac{r - f(a)}{r - g(a)} = 1$$

証明サイズ：コミット2個、評価値6個。

---

### ガジェット5：Prescribed Permutation Check（指定置換確認）

**目標：** 指定された置換 $W: \Omega \to \Omega$ に対して

$$f(y) = g(W(y)) \quad \forall y \in \Omega$$

を証明する。

**例（$k = 3$）：**
$$W(\omega^0) = \omega^2, \quad W(\omega^1) = \omega^0, \quad W(\omega^2) = \omega^1$$

**単純アプローチの問題点：**

$f(y) - g(W(y)) = 0$ をZeroTestで証明しようとすると、$f(y) - g(W(y))$ は**次数 $k^2$** の多項式になってしまいます（$W$ が次数 $k$ の多項式なので）。
→ 証明者が次数 $k^2$ の多項式を扱う必要 → 二乗時間がかかる！

**解決策：** 次数 $2k$ の積確認に帰着させる

**観察：**

$$\left(W(a), f(a)\right)_{a \in \Omega} \text{ が } \left(a, g(a)\right)_{a \in \Omega} \text{ の置換}$$

$$\implies f(y) = g(W(y)) \quad \forall y \in \Omega$$

**例で確認：**

| | 左タプル | 右タプル |
|---|---|---|
| $a = \omega^0$ | $(\omega^2, f(\omega^0))$ | $(\omega^0, g(\omega^0))$ |
| $a = \omega^1$ | $(\omega^0, f(\omega^1))$ | $(\omega^1, g(\omega^1))$ |
| $a = \omega^2$ | $(\omega^1, f(\omega^2))$ | $(\omega^2, g(\omega^2))$ |

左は $((\omega^2, 5), (\omega^0, 5), (\omega^1, 6))$、右は $((\omega^0, 5), (\omega^1, 6), (\omega^2, 11))$ のように、同じペアの並べ替えになっています。

**二変数多項式を使った証明：**

$$\hat{f}(X, Y) \coloneqq \prod_{a \in \Omega} (X - Y \cdot W(a) - f(a))$$

$$\hat{g}(X, Y) \coloneqq \prod_{a \in \Omega} (X - Y \cdot a - g(a))$$

（どちらも総次数 $k$ の二変数多項式）

**補題（一意因子分解域の性質より）：**

$$\hat{f}(X, Y) = \hat{g}(X, Y) \iff \left(W(a), f(a)\right)_{a \in \Omega} \text{ は } \left(a, g(a)\right)_{a \in \Omega} \text{ の置換}$$

**完全プロトコル：**

1. 検証者が $r, s \xleftarrow{\$} \mathbb{F}_p$ をランダムに選ぶ
2. Schwartz-Zippelにより：$\hat{f}(r, s) = \hat{g}(r, s) \implies \hat{f}(X, Y) = \hat{g}(X, Y)$ （高確率）
3. 証明者が積確認（ProdCheck）を実行：

$$\prod_{a \in \Omega} \frac{r - s \cdot W(a) - f(a)}{r - s \cdot a - g(a)} = 1$$

**定理：** このプロトコルは完全かつ健全（$2d/p$ が無視できる場合）。

### ガジェット階層のまとめ

```
多項式等値テスト
    ↓
ZeroTest on Ω
    ↓
SumCheck / ProdCheck
    ↓
Permutation Check
    ↓
Prescribed Permutation Check  ← PLONKで使用！
```

---

## 9. PLONK：汎用回路のためのSNARK

### PLONKとは

**PLONK**（Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge）は、任意の算術回路に対するSNARKです（論文: eprint/2019/953）。

### PLONKが使われている理由

PLONKはIOP（対話型オラクル証明）として設計されており、様々な多項式コミットメントスキームと組み合わせられます：

| 組み合わせ | システム | 特徴 |
|---|---|---|
| PLONK IOP + KZG (ペアリング) | Aztec, JellyFish | 高速 |
| PLONK IOP + Bulletproofs | Halo2 | 信頼されたセットアップ不要・検証遅 |
| PLONK IOP + FRI (ハッシュ) | Plonky2 | 信頼されたセットアップ不要 |

### ステップ1：計算トレースの構築

**算術回路とは？** 加算ゲートと乗算ゲートからなる回路。例：

$$C(x_1, x_2, w_1) = (x_1 + x_2)(x_2 + w_1)$$

入力 $x_1 = 5, x_2 = 6, w_1 = 1$ の場合：

```
ゲート0（+）: 左入力=5, 右入力=6, 出力=11
ゲート1（+）: 左入力=6, 右入力=1, 出力=7
ゲート2（×）: 左入力=11, 右入力=7, 出力=77
```

これを**計算トレース（Computation Trace）**と呼びます。

### ステップ2：トレースの多項式エンコード

**サイズの定義：**
- $|C|$ = 回路のゲート数
- $|I| = |I_x| + |I_w|$ = 公開入力 + 秘密入力（witness）の数
- $d \coloneqq 3|C| + |I|$（例では $d = 3 \times 3 + 3 = 12$）
- $\Omega \coloneqq \{1, \omega, \omega^2, \ldots, \omega^{d-1}\}$（$d$ 個の点）

**エンコードルール：**

証明者は多項式 $T \in \mathbb{F}_p^{(\leq d)}[X]$ を補間して：

- **入力のエンコード：** $T(\omega^{-j}) = \text{入力 } \#j \quad (j = 1, \ldots, |I|)$
- **ゲートのエンコード：** $\forall l = 0, \ldots, |C|-1$：
  - $T(\omega^{3l})$：ゲート $\#l$ の左入力
  - $T(\omega^{3l+1})$：ゲート $\#l$ の右入力
  - $T(\omega^{3l+2})$：ゲート $\#l$ の出力

**具体例（$d = 12$）：**

| 点 | 値 | 意味 |
|---|---|---|
| $T(\omega^{-1})$ | $5$ | 入力 $x_1$ |
| $T(\omega^{-2})$ | $6$ | 入力 $x_2$ |
| $T(\omega^{-3})$ | $1$ | 入力 $w_1$ |
| $T(\omega^0)$ | $5$ | ゲート0の左入力 |
| $T(\omega^1)$ | $6$ | ゲート0の右入力 |
| $T(\omega^2)$ | $11$ | ゲート0の出力 |
| $T(\omega^3)$ | $6$ | ゲート1の左入力 |
| $T(\omega^4)$ | $1$ | ゲート1の右入力 |
| $T(\omega^5)$ | $7$ | ゲート1の出力 |
| $T(\omega^6)$ | $11$ | ゲート2の左入力 |
| $T(\omega^7)$ | $7$ | ゲート2の右入力 |
| $T(\omega^8)$ | $77$ | ゲート2の出力 |

証明者はFFT（高速フーリエ変換）で $T(X)$ の係数を $O(d \log d)$ 時間で計算できます。

### ステップ3：トレースの正しさの証明

証明者は以下の4つの性質を証明する必要があります：

#### 性質(1)：正しい入力のエンコード

両者が $x$-入力から多項式 $v(X) \in \mathbb{F}_p^{(\leq |I_x|)}[X]$ を補間します：

$$v(\omega^{-j}) = \text{入力 } \#j \quad (j = 1, \ldots, |I_x|)$$

証明者は $\Omega_{\text{inp}} \coloneqq \{\omega^{-1}, \omega^{-2}, \ldots, \omega^{-|I_x|}\}$ 上での**ZeroTest**で：

$$T(y) - v(y) = 0 \quad \forall y \in \Omega_{\text{inp}}$$

を証明します。

#### 性質(2)：各ゲートの正しい評価

**セレクター多項式 $S(X)$** を定義します：

$$S(\omega^{3l}) = \begin{cases} 1 & \text{ゲート } \#l \text{ が加算ゲート} \\ 0 & \text{ゲート } \#l \text{ が乗算ゲート} \end{cases}$$

加算ゲートの正しさ：$T(\omega^{3l}) + T(\omega^{3l+1}) = T(\omega^{3l+2})$
乗算ゲートの正しさ：$T(\omega^{3l}) \cdot T(\omega^{3l+1}) = T(\omega^{3l+2})$

これを統合すると、$\forall y \in \Omega_{\text{gates}} \coloneqq \{1, \omega^3, \omega^6, \ldots, \omega^{3(|C|-1)}\}$ において：

$$S(y) \cdot [T(y) + T(\omega y)] + (1 - S(y)) \cdot T(y) \cdot T(\omega y) = T(\omega^2 y)$$

証明者は $\Omega_{\text{gates}}$ 上での**ZeroTest**でこれを証明します：

$$S(y) \cdot [T(y) + T(\omega y)] + (1 - S(y)) \cdot T(y) \cdot T(\omega y) - T(\omega^2 y) = 0$$

**セットアップ：**

$$\text{Setup}(C) \to pp \coloneqq S,\quad vp \coloneqq \boxed{S}$$

セレクター多項式はオフライン（回路固定後）に一度だけ計算します。

#### 性質(3)：配線制約（Wiring Constraints）

回路の「配線」とは、あるゲートの出力が別のゲートの入力になること、または入力が複数のゲートで使われることです。

**例：**
- $T(\omega^{-2}) = T(\omega^1) = T(\omega^3)$（$x_2 = 6$ が複数の場所で使われる）
- $T(\omega^{-1}) = T(\omega^0)$（$x_1 = 5$ がゲート0の左入力）
- $T(\omega^2) = T(\omega^6)$（ゲート0の出力がゲート2の左入力）
- $T(\omega^{-3}) = T(\omega^4)$（$w_1 = 1$ がゲート1の右入力）

これは**回転置換 $W: \Omega \to \Omega$** として表現できます：

$$W(\omega^{-2}, \omega^1, \omega^3) = (\omega^1, \omega^3, \omega^{-2}), \quad W(\omega^{-1}, \omega^0) = (\omega^0, \omega^{-1}), \ldots$$

**補題：**

$$\forall y \in \Omega: T(y) = T(W(y)) \implies \text{配線制約は満たされる}$$

証明者は**指定置換確認（Prescribed Permutation Check）**を使います：

$$\forall y \in \Omega: T(y) - T(W(y)) = 0$$

#### 性質(4)：出力が0

最後のゲートの出力が0であることを確認（回路が受理状態）：

$$T(\omega^{3|C|-1}) = 0$$

これは点評価の確認なので簡単です。

---

## 10. PLONKの完全プロトコル

```
Setup(C) → pp := (S, W),  vp := (⌈S⌉ and ⌈W⌉)  [信頼不要]

証明者 P(pp, x, w)                    検証者 V(vp, x)

T(X) ∈ F_p^(≤d)[X] を構築           v(X) ∈ F_p^(≤|I_x|)[X] を構築
         ──── ⌈T⌉ ────────────────────▶
  (以下を証明)

(1) ゲート制約（ZeroTest on Ω_gates）:
    S(y)·[T(y) + T(ωy)] + (1-S(y))·T(y)·T(ωy) - T(ω²y) = 0

(2) 入力制約（ZeroTest on Ω_inp）:
    T(y) - v(y) = 0

(3) 配線制約（Prescribed Permutation Check on Ω）:
    T(y) - T(W(y)) = 0

(4) 出力制約（点評価）:
    T(ω^{3|C|-1}) = 0
```

**定理（eprint/2019/953）：** PLONK Poly-IOPは完全かつ知識健全（knowledge sound）。$7|C|/p$ が無視できる場合。

**性能：**
- 証明：定数個のコミットメント（$O(1)$）
- 証明者時間：$O(|C| \log |C|)$
- 検証者時間：$O(|x| + \log |C|)$（非常に速い）

---

### PLONKishアリスメタイゼーション（一般化）

PLONKは加算・乗算以外のゲートにも拡張できます。

**Plonkishの計算トレース**（AIRとも呼ばれる）は多列のテーブル：

| $u$ | $v$ | $w$ | $t$ | $r$ | $s$ |
|---|---|---|---|---|---|
| $u_1$ | $v_1$ | $w_1$ | $t_1$ | $r_1$ | $s_1$ |
| $u_2$ | $v_2$ | $w_2$ | $t_2$ | $r_2$ | $s_2$ |
| $\vdots$ | | | | | |

**カスタムゲートの例：**

$$\forall y \in \Omega_{\text{gates}}: v(y\omega) + w(y) \cdot t(y) - t(y\omega) = 0$$

**Plookup：** ある値が事前に定義されたリストに含まれることを証明する拡張（例：ビット範囲チェック）。

---

## 11. まとめ

### 全体の流れの復習

```
任意の計算 C(x, w)
    ↓ 算術化（Arithmetization）
計算トレース（テーブル）
    ↓ 多項式エンコード
多項式 T(X)
    ↓ Poly-IOP
多項式等値テスト・ZeroTest・ProdCheck・Permutation Check
    ↓ KZG などの PCS でコンパイル
SNARK（短い証明）
```

### 各コンポーネントの役割

| コンポーネント | 役割 |
|---|---|
| **KZG PCS** | 多項式を「封筒に入れる」＋点評価の証明 |
| **Poly-IOP** | 多項式の性質を対話的に証明するフレームワーク |
| **ZeroTest** | ある集合上で多項式がゼロであることを証明 |
| **ProdCheck** | 積が1であることを証明 |
| **Permutation Check** | 値列が置換関係にあることを証明 |
| **PLONK** | 回路のゲート・入力・配線の正しさを多項式で表現して証明 |

### 安全性の根拠

| 仮定 | 内容 |
|---|---|
| **離散対数問題** | 楕円曲線上での逆算が困難 |
| **q-SDH（q-Strong Diffie-Hellman）** | KZGの束縛性の根拠 |
| **Schwartz-Zippel補題** | ランダム点での評価が多項式等値テストに有効 |

### 実世界での応用

- **Ethereum（zkEVM）：** スマートコントラクトの実行の正しさをゼロ知識で証明
- **プライバシーコイン（Zcash など）：** 残高を隠したまま送金の正当性を証明
- **zkRollup：** Layer 2のトランザクションをLayer 1で効率的に検証

---

> **参考文献：**
> - Gabizon, Williamson, Ciobotaru, "PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge," 2019. (eprint/2019/953)
> - Kate, Zaverucha, Goldberg, "Constant-Size Commitments to Polynomials and Their Applications," 2010.
> - Lee, "Dory: Efficient, Transparent arguments for Generalised Inner Products and Polynomial Commitments," 2021. (eprint/2020/1274)
