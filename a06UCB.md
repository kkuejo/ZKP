# ゼロ知識証明 講義6：ペアリングと離散対数に基づく多項式コミットメント

> **対象読者**: 数学・暗号理論の基礎知識がない方でも理解できるよう、概念を順番に丁寧に解説します。

---

## 目次

1. [全体像：SNARKとは何か？](#1-全体像snarkとは何か)
2. [多項式コミットメントとは何か？](#2-多項式コミットメントとは何か)
3. [数学的な背景知識](#3-数学的な背景知識)
   - 3.1 [群（Group）とは](#31-群groupとは)
   - 3.2 [生成元（Generator）とは](#32-生成元generatorとは)
   - 3.3 [離散対数問題](#33-離散対数問題)
   - 3.4 [Diffie-Hellman仮定](#34-diffie-hellman仮定)
   - 3.5 [双線型ペアリング（Bilinear Pairing）](#35-双線型ペアリングbilinear-pairing)
   - 3.6 [BLS署名の例](#36-bls署名の例)
4. [KZG多項式コミットメント](#4-kzg多項式コミットメント)
   - 4.1 [鍵生成（Keygen）](#41-鍵生成keygen)
   - 4.2 [コミット（Commit）](#42-コミットcommit)
   - 4.3 [評価（Eval）](#43-評価eval)
   - 4.4 [検証（Verify）](#44-検証verify)
   - 4.5 [健全性（Soundness）の証明](#45-健全性soundnessの証明)
   - 4.6 [知識健全性とKoE仮定](#46-知識健全性とkoe仮定)
   - 4.7 [一般群モデル（GGM）](#47-一般群モデルggm)
   - 4.8 [KZGの性能まとめ](#48-kzgの性能まとめ)
   - 4.9 [セレモニー（Ceremony）：信頼済みセットアップの分散化](#49-セレモニーceremony信頼済みセットアップの分散化)
5. [KZGの拡張バリアント](#5-kzgの拡張バリアント)
   - 5.1 [多変数多項式コミットメント](#51-多変数多項式コミットメント)
   - 5.2 [ゼロ知識化](#52-ゼロ知識化)
   - 5.3 [バッチ開示：単一多項式の複数点](#53-バッチ開示単一多項式の複数点)
   - 5.4 [バッチ開示：複数多項式の複数点](#54-バッチ開示複数多項式の複数点)
   - 5.5 [Plonk への応用](#55-plonk-への応用)
   - 5.6 [vSQL / Libra への応用](#56-vsql--libra-への応用)
6. [離散対数に基づく多項式コミットメント](#6-離散対数に基づく多項式コミットメント)
   - 6.1 [Bulletproofs の基本アイデア](#61-bulletproofs-の基本アイデア)
   - 6.2 [Bulletproofs の詳細プロトコル](#62-bulletproofs-の詳細プロトコル)
   - 6.3 [Bulletproofs の性能](#63-bulletproofs-の性能)
   - 6.4 [Hyrax](#64-hyrax)
   - 6.5 [Dory](#65-dory)
   - 6.6 [Dark](#66-dark)
7. [全スキームの比較まとめ](#7-全スキームの比較まとめ)

---

## 1. 全体像：SNARKとは何か？

### SNARKって何？

**SNARK**（Succinct Non-interactive ARgument of Knowledge：簡潔な非対話型知識論証）とは、

> 「私はある計算の答えを知っている」ということを、**答えを明かさずに**、しかも**非常に短い証明**で相手に納得させる技術

です。

たとえば、「私はこのパズルの解き方を知っている」と主張したいとき、解き方を全部教えてしまえば簡単ですが、それでは秘密が漏れます。SNARKは「解き方を教えずに、確かに知っていると証明する」方法です。

### SNARKの構成方法（ブレンダーの比喩）

この講義では、SNARKを作るために2つの部品を「ブレンダーで混ぜる」比喩が使われています：

```
[多項式コミットメント] + [多項式IOP] → SNARK（一般回路向け）
```

- **多項式IOP**（Polynomial Interactive Oracle Proof）：証明のプロトコルの骨格
- **多項式コミットメント**（Polynomial Commitment Scheme）：多項式を「封印」して、後から正直に開いたことを証明する仕組み

今回の講義は、この「多項式コミットメント」に焦点を当てています。

---

## 2. 多項式コミットメントとは何か？

### 直感的な説明

「多項式コミットメント」を日常の例で説明します：

1. **コミット（封印）**：あなたが答え $y = f(x)$ を書いた紙を封筒に入れ、封をして相手に渡します。
2. **チャレンジ**：相手が「$x = 5$ のときの値を見せて」と要求します。
3. **証明**：あなたが封筒を開け、$f(5)$ の値と、それが確かに最初に封印した多項式から来たことの証明を見せます。

### フォーマルなプロトコル図

```
証明者 (Prover)             検証者 (Verifier)
  f(x) を持っている
       
  commit(f) → com_f ─────────→ com_f を受け取る
  
       ←─────────────────────── u（評価点）を送る
       
  v = f(u) を計算
  証明 π を計算
  (v, π) ──────────────────→ (com_f, u, v, π) を検証
                                 → accept / reject
```

- $f(x)$：証明者が持っている多項式（秘密）
- $com_f$：コミットメント（封印）
- $u$：検証者が選ぶ評価点
- $v$：$f(u)$ の値（証明者が主張）
- $\pi$：$f(u) = v$ であることの証明

### 多項式コミットメントの定義（4つのアルゴリズム）

| アルゴリズム | 説明 |
|---|---|
| $\text{keygen}(\lambda, \mathcal{F}) \to gp$ | セキュリティパラメータと多項式族 $\mathcal{F}$ から公開パラメータ $gp$ を生成 |
| $\text{commit}(gp, f) \to com_f$ | 多項式 $f$ を公開パラメータ $gp$ でコミット |
| $\text{eval}(gp, f, u) \to v, \pi$ | 点 $u$ での評価値 $v$ と証明 $\pi$ を生成 |
| $\text{verify}(gp, com_f, u, v, \pi) \to \text{accept/reject}$ | 証明を検証 |

### 知識健全性（Knowledge Soundness）

重要な性質：悪意ある証明者が「$f(u) = v$」という嘘をついても、検証者を騙せないこと。

より正確には：検証をパスできる証明者からは、実際の多項式 $f$ を**抽出**できるような抽出器 $E$ が存在しなければならない。つまり証明者は「本当に $f$ を知っている」。

---

## 3. 数学的な背景知識

KZGなどの方式を理解するためには、いくつかの数学概念が必要です。順番に説明します。

### 3.1 群（Group）とは

**群**とは、ある集合 $\mathbb{G}$ と演算 $*$ の組で、以下の4つの性質を満たすものです：

| 性質 | 説明 |
|---|---|
| **閉じている（Closure）** | $a, b \in \mathbb{G}$ なら $a * b \in \mathbb{G}$ |
| **結合法則（Associativity）** | $(a * b) * c = a * (b * c)$ |
| **単位元（Identity）** | ある $e \in \mathbb{G}$ が存在して、$e * a = a * e = a$ |
| **逆元（Inverse）** | 各 $a$ に対して $b$ が存在して、$a * b = b * a = e$ |

**具体例**：
- 整数 $\{\ldots, -2, -1, 0, 1, 2, \ldots\}$ と加法 $+$
- 素数 $p$ に対する $\{1, 2, \ldots, p-1\}$ と乗法 $\times \pmod{p}$
- **楕円曲線**（暗号で最もよく使われる）

### 3.2 生成元（Generator）とは

ある群 $\mathbb{G}$ において、元 $g$ の**べき乗をすべて取ると群全体が得られる**とき、$g$ を**生成元**と言います。

**具体例**：$\mathbb{Z}_7^* = \{1, 2, 3, 4, 5, 6\}$（mod 7 の乗法群）

$$3^1 = 3, \quad 3^2 = 9 \equiv 2, \quad 3^3 = 27 \equiv 6$$
$$3^4 = 81 \equiv 4, \quad 3^5 \equiv 5, \quad 3^6 \equiv 1 \pmod{7}$$

よって $g = 3$ はこの群の生成元です。$\{3, 2, 6, 4, 5, 1\}$ = $\{1, 2, 3, 4, 5, 6\}$ — 全元が生成されています。

### 3.3 離散対数問題

群 $\mathbb{G}$ の生成元 $g$ を使うと、任意の元を $\{g, g^2, g^3, \ldots, g^{p-1}\}$ と表現できます。

**離散対数問題（DLP）**：$y \in \mathbb{G}$ が与えられたとき、$g^x = y$ を満たす $x$ を求めよ。

**例**：$3^x \equiv 4 \pmod{7}$ を解け → $x = 4$（上の計算から）

**離散対数仮定**：この計算は、大きな群に対しては**計算困難**（現実的な時間では解けない）と仮定します。

> **日常の例え**：時計の針を何回か回転させた後の位置は簡単に計算できますが、「最終的にこの位置になるまで何回転させたか」を逆に求めるのは難しい、というイメージです。

### 3.4 Diffie-Hellman仮定

**計算的Diffie-Hellman仮定（CDH仮定）**：

$$\mathbb{G}, g, g^x, g^y \text{ が与えられても、} g^{xy} \text{ は計算できない}$$

つまり、$g^x$ と $g^y$ を見ても、掛け合わせた指数 $xy$ を指数部分で計算することはできない、ということです。

### 3.5 双線型ペアリング（Bilinear Pairing）

**ペアリング**は、2つの群の元を掛け合わせてターゲット群の元を得る特殊な関数です。

設定：$(p, \mathbb{G}, g, \mathbb{G}_T, e)$

- $\mathbb{G}$：ベース群（位数 $p$、生成元 $g$）
- $\mathbb{G}_T$：ターゲット群（位数 $p$）
- $e$：ペアリング関数 $e: \mathbb{G} \times \mathbb{G} \to \mathbb{G}_T$

**最重要性質（双線型性）**：

$$e(P^x, Q^y) = e(P, Q)^{xy}$$

**具体例**：

$$e(g^x, g^y) = e(g, g)^{xy} = e(g^{xy}, g)$$

**なぜ便利か**：$g^x$ と $g^y$ だけ知っていても、ペアリングを使えば「ある元が $g^{xy}$ であるかどうか」を $x$ や $y$ を知らずに確認できます！

$$e(g^x, g^y) \stackrel{?}{=} e(h, g) \iff h = g^{xy}$$

### 3.6 BLS署名の例

ペアリングの応用として BLS署名（Boneh–Lynn–Shacham, 2001）があります：

- **鍵生成**：秘密鍵 $x$、公開鍵 $g^x$
- **署名**：$\sigma = H(m)^x$（$H$ はメッセージを群の元にマップするハッシュ関数）
- **検証**：

$$e(H(m), g^x) \stackrel{?}{=} e(\sigma, g)$$

ペアリングにより、秘密鍵 $x$ を知らなくても署名の正しさを検証できます。

---

## 4. KZG多項式コミットメント

KZGは **Kate–Zaverucha–Goldberg (2010)** が提案した方式で、現在最も広く使われている多項式コミットメントです。Ethereum の EIP-4844（blob トランザクション）でも使われています。

### 4.1 鍵生成（Keygen）

**設定**：双線型群 $(p, \mathbb{G}, g, \mathbb{G}_T, e)$ と、多項式族 $\mathcal{F} = \mathbb{F}_p^{(\leq d)}[X]$（次数 $\leq d$ のすべての多項式）

**手順**：
1. ランダムな $\tau \in \mathbb{F}_p$ をサンプル（秘密のトラップドア）
2. 公開パラメータを計算：

$$gp = \left(g, g^\tau, g^{\tau^2}, \ldots, g^{\tau^d}\right)$$

3. $\tau$ を**削除**！（信頼済みセットアップ）

> **重要**：$\tau$ が漏れると偽の証明が作れてしまいます。誰も $\tau$ を知らない状態にする必要があります（後述の「セレモニー」で対処）。

**なぜ $g^{\tau^i}$ が必要か**：実際の $\tau$ の値を知らなくても、多項式を「$\tau$ で評価した結果の群元」を計算できるようにするためです。

### 4.2 コミット（Commit）

多項式 $f(x) = f_0 + f_1 x + f_2 x^2 + \cdots + f_d x^d$ に対して：

$$com_f = g^{f(\tau)} = g^{f_0 + f_1\tau + f_2\tau^2 + \cdots + f_d\tau^d}$$

$$= g^{f_0} \cdot (g^\tau)^{f_1} \cdot (g^{\tau^2})^{f_2} \cdots (g^{\tau^d})^{f_d}$$

**なぜこれで「コミット」になるか**：
- $gp$ の各要素を組み合わせることで計算できる（$\tau$ の値は不要）
- $com_f$ から $f$ の係数を逆算することは離散対数仮定により不可能
- 後で「別の多項式のコミットメントだった」と主張することも不可能

コミットは **1つの群の元**（約32バイト）— とてもコンパクトです！

### 4.3 評価（Eval）

証明者は点 $u$ でのコミットメントを「開示」します。

**核心アイデア**：$u$ が $f(x) - f(u)$ の根である（$f(u) - f(u) = 0$）ことから：

$$f(x) - f(u) = (x - u) \cdot q(x)$$

と書ける商多項式 $q(x)$ が必ず存在します（多項式の因数分解）。

**手順**：
1. $v = f(u)$ を計算
2. $q(x) = \frac{f(x) - f(u)}{x - u}$ を計算（多項式の割り算）
3. 証明 $\pi = g^{q(\tau)}$ を $gp$ を使って計算

### 4.4 検証（Verify）

検証者は $(gp, com_f, u, v, \pi)$ を受け取ります。

**チェックしたいこと**：

$$f(\tau) - f(u) = (\tau - u) \cdot q(\tau)$$

が成立しているか（これが成立 $\iff$ $f(u) = v$ が正しい）。

**問題**：$\tau$ は誰も知らない！ $g^{\tau-u}$ と $g^{q(\tau)}$ しか持っていない。

**解決策**：ペアリングを使う！

$$e\!\left(\frac{com_f}{g^v},\ g\right) \stackrel{?}{=} e\!\left(g^{\tau - u},\ \pi\right)$$

**左辺を展開**：

$$e\!\left(g^{f(\tau) - v},\ g\right) = e(g, g)^{f(\tau) - v}$$

**右辺を展開**：

$$e\!\left(g^{\tau - u},\ g^{q(\tau)}\right) = e(g, g)^{(\tau - u) \cdot q(\tau)}$$

正直な証明者なら $v = f(u)$ なので $f(\tau) - v = f(\tau) - f(u) = (\tau - u) q(\tau)$ となり、等式が成立します。

**完全なプロトコル図**：

```
        gp = (g, g^τ, g^τ², ..., g^τ^d)
        多項式の次数 ≤ d

証明者                          検証者
f(x) を持つ

com_f = g^{f(τ)} ─────────────→ com_f を受け取る

         ←────────────────────── u を送る

f(x) - f(u) = (x-u)q(x) を計算
π = g^{q(τ)} を計算
(v, π) ──────────────────────→ e(com_f/g^v, g) = e(g^{τ-u}, π)
                                 を検証
```

### 4.5 健全性（Soundness）の証明

**q-SBDH仮定**（q-Strong Bilinear Diffie-Hellman）：

$(p, \mathbb{G}, g, \mathbb{G}_T, e)$ と $(g, g^\tau, g^{\tau^2}, \ldots, g^{\tau^d})$ が与えられても、任意の $u$ に対して $e(g, g)^{\frac{1}{\tau - u}}$ を計算することはできない。

**背理法による証明**：

$v^* \neq f(u)$ であるにもかかわらず、$\pi^*$ が検証をパスしたと仮定する。

$$e\!\left(\frac{com_f}{g^{v^*}},\ g\right) = e\!\left(g^{\tau - u},\ \pi^*\right)$$

$\delta = f(u) - v^* \neq 0$ とおくと：

$$e\!\left(g^{(\tau-u)q(\tau) + \delta},\ g\right) = e\!\left(g^{\tau-u},\ \pi^*\right)$$

整理すると：

$$e(g, g)^{\frac{\delta}{\tau - u}} = e\!\left(g, \frac{\pi^*}{g^{q(\tau)}}\right)$$

これは $e(g,g)^{\frac{1}{\tau-u}}$ を計算できることを意味し、q-SBDH仮定に矛盾。よって偽の証明はパスできない。

### 4.6 知識健全性とKoE仮定

**問題**：証明者が $com_f = g^{f(\tau)}$ を送ってきたとき、「本当に $f$ を知っている」と言えるか？

**知識of指数仮定（KoE仮定）**を使います：

1. セットアップ時に、追加でランダムな $\alpha$ をサンプルし、$g^\alpha, g^{\alpha\tau}, g^{\alpha\tau^2}, \ldots, g^{\alpha\tau^d}$ も公開パラメータに含める
2. コミット時に、証明者は $com_f = g^{f(\tau)}$ と $com_f' = g^{\alpha f(\tau)}$ の**両方**を送る
3. 検証者は $e(com_f, g^\alpha) \stackrel{?}{=} e(com_f', g)$ をチェック

この等式が成立 $\iff$ $com_f$ と $com_f'$ が一致した多項式 $f$ から来ている。

KoE仮定の下で、これをパスできる証明者から実際の $f$ を抽出する「抽出器」が存在します。

### 4.7 一般群モデル（GGM）

KoE仮定の代替として、**一般群モデル（GGM）**があります [Shoup'97, Maurer'05]。

**直感**：「群の演算だけしかできない」という制限の下で敵対者を考えるモデル。

$g, g^\tau, g^{\tau^2}, \ldots, g^{\tau^d}$ が与えられたとき、敵対者は**その線形結合**しか計算できない。

GGMを使うことで：
- KoE仮定が不要になる
- コミットメントのサイズが小さくなる（$com_f'$ が不要）

詳細は Dan Boneh と Victor Shoup の教科書 "A Graduate Course in Applied Cryptography" §16.3 を参照。

### 4.8 KZGの性能まとめ

| フェーズ | コスト |
|---|---|
| **鍵生成（Keygen）** | $O(d)$ の群演算、**信頼済みセットアップ**が必要 |
| **コミット** | $O(d)$ の群べき乗、コミットサイズは $O(1)$（1群元） |
| **評価（Eval）** | $O(d)$ の群べき乗（$q(x)$ の計算は線形時間で可能） |
| **証明サイズ** | $O(1)$（1群元） |
| **検証時間** | $O(1)$（1回のペアリング計算） |

**KZGの最大の特徴**：証明が1つの群の元（約32バイト）、検証が1回のペアリングだけ — 非常に効率的！

### 4.9 セレモニー（Ceremony）：信頼済みセットアップの分散化

**問題**：$\tau$ を誰も知らない状態で $gp = (g^\tau, g^{\tau^2}, \ldots, g^{\tau^d})$ を生成する必要がある。

**解決策**：多数の参加者が順番に秘密を混ぜる「セレモニー」

**手順**：

1. 初期 $gp = (g_1, g_2, \ldots, g_d)$ から始める
2. 各参加者がランダムな $s$ をサンプルして更新：

$$gp' = \left(g_1^s, g_2^{s^2}, \ldots, g_d^{s^d}\right) = \left(g^{\tau s}, g^{(\tau s)^2}, \ldots, g^{(\tau s)^d}\right)$$

新しいトラップドアは $\tau \cdot s$（参加者だけが $s$ を知る）

3. 更新の正当性を確認：
   - 参加者は $s$ を知っていることを証明（$g_1' = g_1^s$ の知識証明）
   - 更新後の $gp'$ が連続したべき乗になっている：$e(g_i', g_1') = e(g_{i+1}', g)$ かつ $g_1' \neq 1$

**安全性**：参加者のうち**少なくとも1人が正直で自分の秘密を破棄**すれば、$\tau$ 全体は誰も知らない。

> **Ethereumの文脈**：EIP-4844のKZGセットアップは世界中の何千人もの参加者が参加したセレモニーで行われました。

---

## 5. KZGの拡張バリアント

### 5.1 多変数多項式コミットメント

[Papamanthou-Shi-Tamassia'13]

例：$f(x_1, \ldots, x_k) = x_1 x_3 + x_1 x_4 x_5 + x_7$

**核心アイデア**（多変数版の因数分解）：

$$f(x_1, \ldots, x_k) - f(u_1, \ldots, u_k) = \sum_{i=1}^{k} (x_i - u_i) q_i(\vec{x})$$

**プロトコル**：

- **Keygen**：$\tau_1, \ldots, \tau_k$ をサンプル、$gp$ はすべての単項式 $g^{\text{（各変数の積）}}$ を含む（多線型多項式なら $2^k$ 個の単項式）
- **Commit**：$com_f = g^{f(\tau_1, \ldots, \tau_k)}$
- **Eval**：各 $i$ について $\pi_i = g^{q_i(\vec{\tau})}$ を計算
- **Verify**：

$$e\!\left(\frac{com_f}{g^v},\ g\right) = \prod_{i=1}^{k} e\!\left(g^{\tau_i - u_i},\ \pi_i\right)$$

**性能**：$O(\log N)$ の証明サイズと検証時間（$N$ は変数数）

### 5.2 ゼロ知識化

[ZGKPP'2018]

**問題**：普通のKZGはゼロ知識ではない。$com_f = g^{f(\tau)}$ は決定論的なので、同じ多項式は必ず同じコミットメントになる。

**解決策**：ランダムマスキング

- **コミット**：$com_f = g^{f(\tau) + r\eta}$（$r$ はランダム、$\eta = g$ の追加の秘密ベース）
- **Eval**：

$$f(x) + ry - f(u) = (x - u) q(x) + r'y + y(r - r'(x-u))$$

- 証明 $\pi = (g^{q(\tau) + r'\eta},\ g^{r - r'(\tau - u)})$

ランダム性を混ぜることで、コミットメントから $f$ の情報が漏れなくなります。

### 5.3 バッチ開示：単一多項式の複数点

証明者が1つの多項式 $f$ の複数の点 $u_1, \ldots, u_m$（$m < d$）での値を一括証明したい場合。

**核心アイデア**：

$f(u_1), \ldots, f(u_m)$ から内挿多項式 $h(x)$（$f$ と各点で値が一致する $m-1$ 次多項式）を求めると：

$$f(x) - h(x) = \prod_{i=1}^{m}(x - u_i) \cdot q(x)$$

- 証明：$\pi = g^{q(\tau)}$
- 検証：

$$e\!\left(\frac{com_f}{g^{h(\tau)}},\ g\right) = e\!\left(g^{\prod_{i=1}^{m}(\tau - u_i)},\ \pi\right)$$

これにより、$m$ 点の開示を**1つの群元**で証明できます！

### 5.4 バッチ開示：複数多項式の複数点

証明者が $f_i(u_{i,j}) = v_{i,j}$（$i \in [n], j \in [m]$）を一括証明したい場合：

1. 各 $f_i$ について、評価点での内挿多項式 $h_i(x)$ を求める
2. $f_i(x) - h_i(x) = \prod_j (x - u_{i,j}) \cdot q_i(x)$
3. ランダムな線形結合で全 $q_i(x)$ を組み合わせる

これにより複数多項式の複数点を**1回の検証**で確認できます。

### 5.5 Plonk への応用

[Gabizon-Williamson-Ciobotaru'20]

```
[Plonk 多項式IOP] + [単変数 KZG] → 一般回路向け SNARK
```

Plonkは現在最も広く使われているzk-SNARKの一つで、Ethereum の zkEVM などに使われています。

### 5.6 vSQL / Libra への応用

```
[Sumcheck プロトコル / GKR プロトコル] + [多変数 KZG] → 一般回路向け SNARK
```

---

## 6. 離散対数に基づく多項式コミットメント

KZGは非常に効率的ですが、「信頼済みセットアップ」が必要という欠点があります。以下では、セットアップが**透明**（Transparent）、つまり誰でも検証できる方式を紹介します。

### 6.1 Bulletproofs の基本アイデア

[BCCGP'16, BBBPWM'18]

**透明なセットアップ**：公開パラメータをランダムにサンプルするだけ：

$$gp = (g_0, g_1, g_2, \ldots, g_d) \text{ （群の元をランダムに選ぶ）}$$

**Pedersen ベクトルコミットメント**：

$$com_f = g_0^{f_0} \cdot g_1^{f_1} \cdot g_2^{f_2} \cdots g_d^{f_d}$$

これは離散対数仮定の下で隠匿性（hiding）と拘束性（binding）を持ちます。

**基本アイデア**：次数 $d$ の多項式を、より低次の問題に**帰着（再帰）**させる。

**高レベルのアイデア**（次数3の例、実際は次数4→2に半減）：

```
証明者                              検証者
f₀, f₁, f₂, f₃ を持つ
com_f = g₀^f₀ g₁^f₁ g₂^f₂ g₃^f₃ ──→ com_f を受け取る

                      ←────────────── u（評価点）を送る

v = f₀ + f₁u + f₂u² + f₃u³ ───────→ v を受け取る

f(x)を半分に折り畳む
f'₀ = rf₀+f₂, f'₁ = rf₁+f₃

com_f' = g₀'^f₀' g₁'^f₁' ──────────→ より小さい問題に帰着
v' = f'₀ + f'₁ u
```

### 6.2 Bulletproofs の詳細プロトコル

次数4の多項式（係数 $f_0, f_1, f_2, f_3$）の例：

**ステップ1**：証明者が $v = f_0 + f_1 u + f_2 u^2 + f_3 u^3$ を送る

**ステップ2**：証明者が「左半分」と「右半分」の追加コミットを送る

$$L = g_2^{f_0} g_3^{f_1}, \quad R = g_0^{f_2} g_1^{f_3}$$

また：
$$v_L = f_0 + f_1 u, \quad v_R = f_2 + f_3 u$$

よって $v = v_L + v_R \cdot u^2$（検証者は $v$ を2つに分解できる）

**ステップ3**：検証者がランダムな $r$ を送る

**ステップ4**：以下の帰着が成立：

- 新しい係数：$f_0' = rf_0 + f_2,\ f_1' = rf_1 + f_3$
- 新しいコミットメント：

$$com' = L^r \cdot com_f \cdot R^{r^{-1}} = (g_0^{r^{-1}} g_2)^{rf_0+f_2} \cdot (g_1^{r^{-1}} g_3)^{rf_1+f_3}$$

- 新しいベース：$gp' = (g_0^{r^{-1}} g_2,\ g_1^{r^{-1}} g_3)$
- 新しい値：$v' = rv_L + v_R$

これで**次数4の問題が次数2の問題に帰着**されました！これを $\log d$ 回繰り返すと、次数1（定数）になります。

**検証の流れ**：

1. $v = v_L + v_R \cdot u^{d/2}$ を確認
2. ランダムな $r$ を生成
3. $com' = L^r \cdot com_f \cdot R^{r^{-1}}$、$gp'$、$v' = rv_L + v_R$ を計算
4. これを $\log d$ 回再帰的に繰り返す

**Fiat-Shamir変換**でランダムな $r$ をハッシュ関数で生成すれば、非対話型証明になります。

### 6.3 Bulletproofs の性能

| フェーズ | コスト |
|---|---|
| **鍵生成（Keygen）** | $O(d)$、**透明なセットアップ**（trusted setup 不要） |
| **コミット** | $O(d)$ の群べき乗、$O(1)$ のコミットサイズ |
| **Eval** | $O(d)$ の群べき乗 |
| **証明サイズ** | $O(\log d)$（各ラウンドで $L, R$ を送るので $2\log d$ 個の群元） |
| **検証時間** | $O(d)$（各 $\log d$ ラウンドで $gp'$ を更新するコストが累積） |

**KZGとの比較**：
- ✅ 信頼済みセットアップが不要
- ❌ 証明サイズが $O(\log d)$（KZGは $O(1)$）
- ❌ 検証時間が $O(d)$（KZGは $O(1)$）

### 6.4 Hyrax

[Wahby-Tzialla-shelat-Thaler-Walfish'18]

**改善点**：検証時間を $O(d)$ から $O(\sqrt{d})$ に削減

**アイデア**：係数を $\sqrt{d} \times \sqrt{d}$ の**2次元行列**として表現し、行ごとにコミットする

- **証明サイズ**：$O(\sqrt{d})$
- **検証時間**：$O(\sqrt{d})$

### 6.5 Dory

[Lee'2021]

**改善点**：
- 検証時間を $O(\log d)$ に削減
- 証明者時間も $O(\sqrt{d})$ の指数演算 + $O(d)$ のフィールド演算に改善

**核心アイデア**：検証者の構造化された計算（行列積）を、内積ペアリング引数 [BMMTV'2021] を使って証明者に委任する

- **証明サイズ**：$O(\log d)$
- **検証時間**：$O(\log d)$
- **セットアップ**：透明（ペアリングを使うが trusted setup は不要）

### 6.6 Dark

[Bünz-Fisch-Szepieniec'20]

**特徴**：**位数不明の群**（Group of Unknown Order）を使用

- RSA群や整数の理想クラス群などが使われる
- **証明サイズ**：$O(\log d)$
- **検証時間**：$O(\log d)$

---

## 7. 全スキームの比較まとめ

| スキーム | 証明者コスト | 証明サイズ | 検証時間 | Trusted Setup | 暗号的基盤 |
|---|---|---|---|---|---|
| **KZG** | $O(d)$ | $O(1)$ | $O(1)$ | ✅ 必要 | ペアリング |
| **Bulletproofs** | $O(d)$ | $O(\log d)$ | $O(d)$ | ❌ 不要 | 離散対数 |
| **Hyrax** | $O(d)$ | $O(\sqrt{d})$ | $O(\sqrt{d})$ | ❌ 不要 | 離散対数 |
| **Dory** | $O(d)$ | $O(\log d)$ | $O(\log d)$ | ❌ 不要 | ペアリング |
| **Dark** | $O(d)$ | $O(\log d)$ | $O(\log d)$ | ❌ 不要 | 位数不明群 |

### 選び方のガイド

```
信頼済みセットアップが許容できる場合：
  → KZG が最強（O(1) の証明・検証）
  → Plonk、Groth16 など主要 SNARK の基盤

信頼済みセットアップを避けたい場合：
  → 検証時間が許せるなら Bulletproofs（シンプルで離散対数のみ）
  → 検証時間も短くしたいなら Dory または Dark
  → 証明サイズと検証時間のバランスなら Hyrax
```

### DeIn / ZKP プロジェクトへの示唆

Ethereum ベースのアプリケーション（例：Plonk を使った zkRollup）では KZG が標準的に使われています。Ethereum の EIP-4844 で使われている KZG セットアップは既に済んでいるため、Ethereum エコシステムに乗る場合は KZG を使うのが現実的です。

透明なセットアップが必要な場合（例：permissionless な環境）は Bulletproofs や Dory が選択肢になります。

---

## 参考文献

- Kate, Zaverucha, Goldberg (2010). *Constant-Size Commitments to Polynomials and Their Applications.* ASIACRYPT 2010.
- Papamanthou, Shi, Tamassia (2013). *Signatures of Correct Computation.* TCC 2013.
- Boneh, Lynn, Shacham (2001). *Short Signatures from the Weil Pairing.* ASIACRYPT 2001.
- Bünz et al. (2016/2018). *Bulletproofs: Short Proofs for Confidential Transactions and More.*
- Wahby et al. (2018). *Doubly-Efficient zkSNARKs Without Trusted Setup.*
- Lee (2021). *Dory: Efficient, Transparent arguments for Generalised Inner Products.*
- Bünz, Fisch, Szepieniec (2020). *Transparent SNARKs from DARK Compilers.*
- Gabizon, Williamson, Ciobotaru (2020). *PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge.*
- Nikolaenko, Ragsdale, Bonneau, Boneh (2022). *Powers-of-Tau to the People.*
- Boneh, Shoup. *A Graduate Course in Applied Cryptography.* §16.3.
