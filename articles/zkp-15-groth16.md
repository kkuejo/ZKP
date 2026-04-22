---
title: "[ゼロ知識証明入門シリーズ 15/30] Groth16 — 3 点で全てを証明する最速 SNARK"
emoji: "🏆"
type: "tech"
topics:
  - zkp
  - snark
  - groth16
  - pairing
  - zcash
published: false
---

**日付**: 2026年4月22日
**学習内容**: **Groth16** (Jens Groth, 2016) は「**証明サイズがわずか 3 点 (~200 bytes)、検証時間 3 ペアリング (~3 ms)**」という、**現在知られている SNARK の中で最も証明が小さい**プロトコル。Zcash の Sprout/Sapling、Tornado Cash、多くのプライバシープロトコルが採用している。本記事では Groth16 の仕組みを **(1) 前提: QAP + Pairing**、**(2) Setup**、**(3) Proof 生成の3要素 $A, B, C$**、**(4) Verify の3ペアリング等式**、**(5) Knowledge Soundness と KoE 仮定**、**(6) ZK 化（α, β, γ, δ のマスキング）**、**(7) Groth16 の弱点と制約** に沿って導出する。式変形を丁寧に追う。

## 0. 本記事の位置づけ

Article 12 で QAP を、Article 13 で KZG/ペアリングを学んだ。これらを組み合わせて**最小の証明** にするのが Groth16 の発明。

Groth16 の驚異:

- **証明サイズ**: $\mathbb{G}_1 \times \mathbb{G}_2 \times \mathbb{G}_1$ = 3 点、BLS12-381 で $(48 + 96 + 48)$ = 192 bytes
- **検証時間**: ペアリング 3 回 + 公開入力の線形結合
- **Prover 時間**: $O(|C| \log |C|)$、数秒〜数分

これほど小さい証明は現時点で他に類を見ない。ただし:

- **回路ごとに Trusted Setup が必要**（汎用 setup が使えない）
- **量子耐性なし**（ペアリングベース）

このトレードオフを理解した上で使う。

構成:

- **第1章**: Groth16 の設計目標
- **第2章**: 前提と記号
- **第3章**: Setup
- **第4章**: Prover: $A, B, C$ の構成
- **第5章**: Verify: 3 ペアリング等式
- **第6章**: 検証の代数的展開
- **第7章**: Knowledge Soundness
- **第8章**: ZK 化
- **第9章**: 制約と弱点
- **第10章**: Q&A とまとめ

## 1. Groth16 の設計目標

### 1.1 背景

2013 年の Pinocchio、2014 年の libsnark が「実用 SNARK」の礎を築いたが、証明は数KBだった。Groth は「**ペアリング型 SNARK の証明サイズ下限**」に挑み、3 点という最小形式に到達した。

### 1.2 数式の全体像（予告）

**Setup** で CRS（共通参照文字列）$\sigma$ を生成。

**Prover**:

$$
\pi = (A, B, C)
$$

- $A \in \mathbb{G}_1$
- $B \in \mathbb{G}_2$
- $C \in \mathbb{G}_1$

**Verifier**:

$$
e(A, B) = e(\alpha_1, \beta_2) \cdot e(L, \gamma_2) \cdot e(C, \delta_2)
$$

ここで $\alpha_1, \beta_2, \gamma_2, \delta_2$ は Setup 定数、$L$ は公開入力の線形結合。

## 2. 前提と記号

### 2.1 QAP の記号（Article 12 より）

- 制約行列から作った列多項式: $a_j(X), b_j(X), c_j(X) \in \mathbb{F}[X]$
- Witness: $\vec w = (w_0, w_1, \ldots, w_n)$
- 合成多項式: $A(X) = \sum w_j a_j(X)$、同様に $B(X), C(X)$
- 消滅多項式: $Z(X) = \prod_{i=1}^{m}(X - i)$ または $X^m - 1$
- QAP 等式: $A(X) B(X) - C(X) = Z(X) H(X)$

### 2.2 ペアリング記号（Article 8 より）

- $\mathbb{G}_1, \mathbb{G}_2, \mathbb{G}_T$: 位数 $r$ の巡回群
- $g_1, g_2$: 生成元
- $e : \mathbb{G}_1 \times \mathbb{G}_2 \to \mathbb{G}_T$: 双線形ペアリング

### 2.3 公開入力と私的入力

$\vec{w}$ を 2 つに分ける:

- **公開**: $w_0, w_1, \ldots, w_\ell$（$\ell$ 個）
- **私的**: $w_{\ell+1}, \ldots, w_n$

Verifier は公開部分のみ知る。

## 3. Setup

### 3.1 秘密パラメータの生成

ランダムに 5 つのスカラー（「毒」）を選ぶ:

$$
\alpha, \beta, \gamma, \delta, \tau \in \mathbb{F}_r
$$

さらに QAP の次数 $d$ までの $\tau$ のべきを使う。

### 3.2 CRS の構成

**公開 CRS**（Prover も Verifier も使える）:

$$
\text{CRS}_{\text{prover}} = \begin{pmatrix}
g_1^\alpha, g_1^\beta, g_2^\beta, g_2^\gamma, g_2^\delta \\
\{g_1^{\tau^i}\}_{i=0}^{d-1} \\
\{g_2^{\tau^i}\}_{i=0}^{d-1} \\
\{g_1^{(\beta a_j(\tau) + \alpha b_j(\tau) + c_j(\tau))/\gamma}\}_{j \leq \ell} \\
\{g_1^{(\beta a_j(\tau) + \alpha b_j(\tau) + c_j(\tau))/\delta}\}_{j > \ell} \\
\{g_1^{\tau^i Z(\tau)/\delta}\}_{i=0}^{d-2}
\end{pmatrix}
$$

**Verifier CRS**:

$$
\text{CRS}_{\text{verifier}} = (g_1^\alpha, g_2^\beta, g_2^\gamma, g_2^\delta, \{g_1^{L_j}\}_{j \leq \ell})
$$

ここで $L_j = (\beta a_j(\tau) + \alpha b_j(\tau) + c_j(\tau))/\gamma$。

### 3.3 「毒」の破棄

$\alpha, \beta, \gamma, \delta, \tau$ が漏れると**偽証明を作れる**ため、MPC ceremony で破棄する。Zcash の Powers of Tau と同様。

### 3.4 回路固有性

Groth16 の setup は **回路ごと**。QAP の多項式 $a_j, b_j, c_j$ を使うので、回路が変われば CRS も変わる。これは PLONK の universal setup との大きな違い。

## 4. Prover: $A, B, C$ の構成

### 4.1 基本アイデア

Prover は 3 つの楕円曲線点 $A \in \mathbb{G}_1, B \in \mathbb{G}_2, C \in \mathbb{G}_1$ を計算する。それぞれは:

- $A$ は「$A(\tau)$ を $\mathbb{G}_1$ で表現」
- $B$ は「$B(\tau)$ を $\mathbb{G}_2$ で表現」
- $C$ は「残りの項」

具体的には、ランダムなマスキング値 $r, s \in \mathbb{F}_r$ を選んで:

$$
A = g_1^{A(\tau) + \alpha + r \delta}
$$

$$
B = g_2^{B(\tau) + \beta + s \delta}
$$

$$
C = g_1^{\frac{\sum_{j > \ell} w_j (\beta a_j(\tau) + \alpha b_j(\tau) + c_j(\tau)) + H(\tau) Z(\tau)}{\delta} + A(\tau) s + B(\tau) r - r s \delta}
$$

### 4.2 複雑な $C$ の構造

$C$ は以下の要素からなる:

1. **私的 witness の線形結合**: $\sum_{j > \ell} w_j (\beta a_j + \alpha b_j + c_j)/\delta$
2. **QAP 除算項**: $H(\tau) Z(\tau)/\delta$
3. **ZK マスキング**: $A(\tau) s + B(\tau) r - rs \delta$

### 4.3 なぜこの形なのか

直感:

- $A, B$ を作ると自然に「$A \cdot B = \alpha \beta + \alpha B(\tau) + \beta A(\tau) + A(\tau) B(\tau) + \text{マスク項}$」が出る
- 検証等式の右辺と合うよう $C$ を調整

数式の辻褄を合わせるための工夫が詰まっている。

## 5. Verify: 3 ペアリング等式

### 5.1 検証式

Verifier は公開入力 $(w_0, w_1, \ldots, w_\ell)$ を使って次を確認:

$$
e(A, B) = e(g_1^\alpha, g_2^\beta) \cdot e\left(\sum_{j=0}^{\ell} w_j \cdot g_1^{L_j}, g_2^\gamma\right) \cdot e(C, g_2^\delta)
$$

ここで $L_j = (\beta a_j(\tau) + \alpha b_j(\tau) + c_j(\tau))/\gamma$。

### 5.2 左辺の展開

$$
e(A, B) = e(g_1^{A(\tau) + \alpha + r\delta}, g_2^{B(\tau) + \beta + s\delta})
$$

双線形性:

$$
= e(g_1, g_2)^{(A(\tau) + \alpha + r\delta)(B(\tau) + \beta + s\delta)}
$$

展開:

$$
= e(g_1, g_2)^{A(\tau) B(\tau) + \alpha B(\tau) + \beta A(\tau) + \alpha\beta + (\ldots) \delta}
$$

ここで $(\ldots) \delta$ は $\delta$ を因数に含む項の合計（後で $C$ 側と相殺）。

### 5.3 右辺の展開

**第1ペアリング**:

$$
e(g_1^\alpha, g_2^\beta) = e(g_1, g_2)^{\alpha\beta}
$$

**第2ペアリング**:

$$
e(\sum w_j g_1^{L_j}, g_2^\gamma) = e(g_1, g_2)^{\gamma \sum w_j L_j} = e(g_1, g_2)^{\sum w_j (\beta a_j(\tau) + \alpha b_j(\tau) + c_j(\tau))}
$$

公開入力部分 $j = 0, \ldots, \ell$ のみ。

**第3ペアリング**:

$$
e(C, g_2^\delta) = e(g_1, g_2)^{\delta \cdot (C \text{ の指数})}
$$

$C$ の指数を $\delta$ 倍すると、分母の $\delta$ が消える。

### 5.4 全部まとめて相殺

QAP 等式 $A(X) B(X) - C(X) = Z(X) H(X)$ を $X = \tau$ で:

$$
A(\tau) B(\tau) - C(\tau) = Z(\tau) H(\tau)
$$

ここで $C(\tau) = \sum_j w_j c_j(\tau)$。

これを使って、左辺 $-$ 右辺 $= 0$ が成立することが代数的に確認できる。

(証明の完全な式変形は煩雑なので省略。詳細は Groth 論文や Thaler の教科書参照)

## 6. 検証の代数的展開

### 6.1 左辺 $-$ 右辺の差を追う

指数を $X = g_1 g_2$ で見ると:

左辺指数:

$$
(A(\tau) + \alpha + r\delta)(B(\tau) + \beta + s\delta)
$$

右辺指数:

$$
\alpha\beta + \sum_{j \leq \ell} w_j (\beta a_j + \alpha b_j + c_j) + \delta \cdot C(\text{指数})
$$

$C$ の指数を展開:

$$
\delta \cdot C = \sum_{j > \ell} w_j (\beta a_j + \alpha b_j + c_j) + H(\tau) Z(\tau) + \delta (A(\tau) s + B(\tau) r - rs\delta)
$$

### 6.2 等式の成立条件

これらが等しいためには:

$$
A(\tau) B(\tau) = \sum_{j} w_j (\beta a_j + \alpha b_j + c_j) + H(\tau) Z(\tau) + \alpha B(\tau) + \beta A(\tau) + \alpha\beta
$$

（$\delta$ の項を両辺相殺済み）

これを整理すると:

$$
A(\tau) B(\tau) - \sum_j w_j c_j(\tau) = H(\tau) Z(\tau)
$$

すなわち **QAP 等式の $\tau$ 評価**そのもの。正しい witness なら成立、偽なら成立しない（Schwartz-Zippel & q-SBDH）。

## 7. Knowledge Soundness

### 7.1 Knowledge of Exponent 仮定

**KoE 仮定**: 攻撃者が $(g_1, g_1^\alpha)$ を与えられて $(h, h^\alpha)$ を生成できたなら、必ず $h = g_1^c$ を計算していた（つまり $c$ を知っている）。

### 7.2 Groth16 の knowledge soundness

**定理**: KoE + q-SBDH のもとで、Groth16 は knowledge soundness を満たす。

**証明の直感**: もし悪意ある Prover が受理証明を作ったとすると、KoE 仮定から Extractor は証明要素の「対応するスカラー」を得られる。これらのスカラーから QAP の witness $\vec{w}$ を復元可能。

実際の証明は **Algebraic Group Model (AGM)** で、より洗練された形で与えられる（Fuchsbauer-Kiltz-Loss 2018）。

## 8. ZK 化

### 8.1 マスキングの役割

$r, s$ というランダムスカラーで $A, B$ を「攪拌」する。これにより:

- $C_f$ の決定性が破れる
- 同じ witness でも毎回違う証明になる
- 対応する Simulator が構成可能

### 8.2 Perfect Zero-Knowledge?

Groth16 は **Perfect ZK**（情報理論的に情報漏洩ゼロ）。Simulator は $\alpha, \beta, \delta$ を知れば、witness なしで受理証明を偽造できる。これは「Trap Door」と呼ばれ、ZK 性の根拠。

### 8.3 NIZK 構築

Fiat-Shamir 不要。Groth16 は設計段階で **CRS モデルでの NIZK**。CRS が信頼されていれば、ランダム oracle 不要で証明できる。

## 9. 制約と弱点

### 9.1 回路ごとの Setup

最大の制約。回路が決まってから ceremony → 新しい回路ごとにやり直し。PLONK の Universal Setup がこれを解消。

### 9.2 量子耐性なし

ECDLP と q-SBDH に依存。Shor のアルゴリズムで破綻。PQ 要件があれば STARK を選ぶ。

### 9.3 Toxic waste の悪用

$(\alpha, \beta, \gamma, \delta, \tau)$ が漏れると:

- 偽の $H(\tau)$ を作れる
- 偽の witness $\vec w'$ で Verify を通せる

したがって ceremony の信頼性が critical。

### 9.4 Malleability

Groth16 証明は **malleable**（改変可能）。攻撃者が `A' = A + X` など変形しても受理される可能性。Zcash では外部で署名を組み合わせて対策。

### 9.5 Rogue Public Input

公開入力 $w_0, \ldots, w_\ell$ の扱いを間違えると攻撃される。Verifier は確実に CRS 由来の $L_j$ を使う必要。

## 10. Q&A

### Q1: Groth16 より小さい SNARK はある？

**ほぼない**。KZG ベースで 3 点は下限に近い。新しい発想（Nova folding など）でも、最終証明サイズは Groth16 と同等かわずかに大きい。

### Q2: なぜ Zcash は Sapling で BLS12-381 に移行した？

- BN254 の安全性低下 (100 bit)
- BLS12-381 は 128 bit 安全
- FFT ドメインが大きい
- 2018 年当時の最良選択

### Q3: Prover 時間の主な占有は？

- **MSM (Multi-Scalar Multiplication)**: $g^{\sum w_j a_j(\tau)}$ の計算
- **FFT**: $H(X)$ の計算
- 大規模回路では MSM が支配的（> 50%）

### Q4: Zcash 以外の採用例は？

- **Tornado Cash** (mixer)
- **Filecoin** (Proof of Replication / SpaceTime)
- **Aleo** (旧実装、現在は独自 SNARK)
- **Hermez (Polygon zkEVM)** (部分)

### Q5: なぜ $A \in \mathbb{G}_1, B \in \mathbb{G}_2$？

サイズの非対称: $\mathbb{G}_1$ (48 bytes) < $\mathbb{G}_2$ (96 bytes)。**片方を小さい $\mathbb{G}_1$ に寄せる** ことで証明サイズを最小化。Type III ペアリングなので準同型がなく、ハッシュ等で対応する。

### Q6: Recent variant (Marlin, PLONK) との違いは？

- **Groth16**: 回路固有 setup、証明 3 点、verify 3 ペアリング
- **Marlin**: universal setup、証明 5〜8 点
- **PLONK**: universal setup、証明 7〜9 点、custom gate 可能

Groth16 は固定回路で最速だが、柔軟性に欠ける。

## 11. まとめ

### 本記事の要点

1. **Groth16**: 証明 3 点、検証 3 ペアリング、pairing-based SNARK の金字塔
2. 前提: **QAP + ペアリング**
3. Setup: 毒 $(\alpha, \beta, \gamma, \delta, \tau)$ を ceremony で破棄
4. Prover 構成: $A, B, C$ の 3 点にすべてを詰め込む
5. Verify: $e(A, B) = e(\alpha, \beta) \cdot e(L, \gamma) \cdot e(C, \delta)$ の等式
6. Knowledge soundness: KoE + q-SBDH 仮定に帰着
7. ZK 化: ランダム $r, s$ でマスキング、Perfect ZK
8. 弱点: 回路固有 setup、量子耐性なし、malleability

### 次の記事（Article 16）へ

次の記事は **PLONK**、「Universal Setup + Custom Gate」の革新。1 度のセレモニーで複数回路に使える。現代 SNARK の主流。

### 3行サマリ

- **Groth16 = 3 点証明・3 ペアリング検証、SNARK のベンチマーク**
- **QAP + ペアリング + α, β, γ, δ のマスキング**
- **Zcash/Tornado の中核**、Setup の回路固有性が唯一の弱点

---

## 参考文献

- Jens Groth. *On the Size of Pairing-Based Non-interactive Arguments.* EUROCRYPT 2016.
- Bryan Parno et al. *Pinocchio: Nearly Practical Verifiable Computation.* IEEE S&P 2013.
- Eli Ben-Sasson et al. *Zerocash.* IEEE S&P 2014.
- Georg Fuchsbauer, Eike Kiltz, Julian Loss. *The Algebraic Group Model and its Applications.* CRYPTO 2018.
- ZKP MOOC Lecture 9 (UC Berkeley, 2023).
