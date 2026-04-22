---
title: "[ゼロ知識証明入門シリーズ 13/30] 多項式コミットメント (PCS) と KZG — ペアリングで多項式を封じ込める"
emoji: "📦"
type: "tech"
topics:
  - zkp
  - snark
  - kzg
  - commitment
  - pairing
published: false
---

**日付**: 2026年4月22日
**学習内容**: **多項式コミットメント方式 (Polynomial Commitment Scheme, PCS)** は、多項式 $f(X)$ を「短いコミットメント $C_f$」として封じ込め、後で「$f(z) = v$」という主張を**短い証拠**で検証させる仕組み。本記事では特に **KZG (Kate-Zaverucha-Goldberg) コミットメント** を **(1) 設計の動機**、**(2) Setup（trusted setup）**、**(3) Commit / Open / Verify**、**(4) 検証の代数的展開**、**(5) Hiding と Binding**、**(6) ZK-fication（マスキング）**、**(7) バッチ化（複数点・複数多項式）**、**(8) MPC Ceremony**、**(9) 他の PCS との比較** を扱う。KZG は Groth16, PLONK, Marlin などの心臓部。

## 0. 本記事の位置づけ

Article 12 で QAP を「多項式の等式」に落とし込んだ。次は、この多項式を Verifier に**効率よく送る**手段が必要。素朴に多項式全体を送れば $O(m)$ バイトで Succinct 性が崩れる。

そこで **PCS** を使い、多項式を固定サイズ（KZG なら 48 bytes）に圧縮する:

$$
C_f = \text{Commit}(f), \quad |C_f| = O(1)
$$

そして「$f(z) = v$」を**短い証拠 $\pi$** で証明できる。これで Succinct な SNARK が実現する。

構成:

- **第1章**: PCS の要件
- **第2章**: KZG のアイデア
- **第3章**: KZG Setup
- **第4章**: Commit, Open, Verify の数式展開
- **第5章**: 安全性 (Hiding, Binding)
- **第6章**: ZK 化
- **第7章**: バッチ評価
- **第8章**: MPC Ceremony
- **第9章**: 他の PCS との比較
- **第10章**: Q&A とまとめ

## 1. PCS の要件

### 1.1 アルゴリズムのインターフェイス

- **Setup($1^\lambda, d$)**: 公開パラメータ $\text{pp}$ を生成（$d$ は多項式の最大次数）
- **Commit(pp, $f$)**: 多項式 $f$ のコミットメント $C_f$
- **Open(pp, $f, z$)**: $f(z) = v$ と証拠 $\pi$
- **Verify(pp, $C_f, z, v, \pi$)**: 0/1 を返す

### 1.2 性質

- **Correctness**: 正直な Prover なら Verify は 1
- **Binding**: コミット後に $f$ を変えると、検証に失敗する（ほぼ確実に）
- **Hiding**: $C_f$ から $f$ の情報が漏れない（KZG は computationally hiding）
- **Knowledge soundness**: 偽の $\pi$ を作れない

### 1.3 Succinctness

- $|C_f| = O(1)$
- $|\pi| = O(1)$
- Verify time $= O(1)$

KZG は全部満たす。FRI は $|\pi| = O(\log^2 d)$。

## 2. KZG のアイデア

### 2.1 発想の核

「多項式 $f(X)$ を暗号化した形で保持し、評価だけできるようにしたい」

KZG の賢さ: **楕円曲線上で $f$ の値を「秘密点 $\tau$」で評価する**。Prover は $\tau$ を知らないので $f$ を再構成できない。Verifier もペアリングで評価結果の整合性だけ確認。

### 2.2 数式の予告

- Commitment: $C_f = g^{f(\tau)} \in \mathbb{G}_1$
- Opening at $z$: $\pi = g^{q(\tau)}$ ここで $q(X) = \frac{f(X) - v}{X - z}$
- Verification: ペアリング等式 $e(C_f - v \cdot g, h) = e(\pi, h^\tau - z \cdot h)$

### 2.3 記号

- $g, h$: それぞれ $\mathbb{G}_1, \mathbb{G}_2$ の生成元
- $\tau$: 秘密のランダム値 (setup で生成)
- $e : \mathbb{G}_1 \times \mathbb{G}_2 \to \mathbb{G}_T$: ペアリング

## 3. KZG Setup

### 3.1 生成する SRS (Structured Reference String)

ランダムな $\tau \in \mathbb{F}_r$ を選び、以下を公開:

$$
\text{pp} = (g, g^\tau, g^{\tau^2}, \ldots, g^{\tau^d}, h, h^\tau)
$$

$\tau$ 自体は**絶対に秘密**にする（「毒」）。

### 3.2 $\tau$ が漏れると何が起きるか

$\tau$ を知る攻撃者は:

- 任意の $f, g$ について偽の $H = \frac{f - g}{Z}$ を作って通せる
- 偽の Prover として振る舞える

したがって setup の**セレモニー**で $\tau$ を破棄する必要がある。

### 3.3 SRS サイズ

次数 $d$ の多項式を扱うには、$g^{\tau^0}, \ldots, g^{\tau^d}$ の $d+1$ 点が必要。

- BLS12-381 で $\mathbb{G}_1$ 点は 48 bytes 圧縮
- $d = 2^{20}$ なら SRS は $\sim 50$ MB

### 3.4 Universal Setup

PLONK では、**同じ SRS で複数の回路を使える**。これが "Universal Setup"。

- $d$ 以下の任意の多項式に使える
- 1 度セレモニーをすれば再利用可能

Groth16 は回路ごとに別の setup が必要なので、Universal ではない。

## 4. Commit, Open, Verify の数式展開

### 4.1 Commit

多項式 $f(X) = \sum_{i=0}^{d} f_i X^i$ をコミット:

$$
C_f = g^{f(\tau)} = g^{\sum_i f_i \tau^i} = \prod_{i=0}^{d} (g^{\tau^i})^{f_i}
$$

SRS の $g^{\tau^i}$ を使えば、$\tau$ を知らずに計算可能。**MSM (Multi-Scalar Multiplication)** で効率化。

### 4.2 Open: 評価 $v = f(z)$ の証拠

$f(z) = v$ を示したい。これは $f(X) - v$ が $X = z$ で 0 を意味する。因数定理:

$$
f(X) - v = (X - z) \cdot q(X)
$$

ここで $q(X)$ は**商多項式**。次数は $d - 1$ 以下。

Prover は $q(X)$ を計算し、コミット:

$$
\pi = g^{q(\tau)} \in \mathbb{G}_1
$$

### 4.3 Verify: ペアリング等式

以下の等式を Verifier が確認:

$$
e(C_f \cdot g^{-v}, h) \stackrel{?}{=} e(\pi, h^\tau \cdot h^{-z})
$$

**代数的な展開**:

左辺:

$$
e(C_f \cdot g^{-v}, h) = e(g^{f(\tau)} \cdot g^{-v}, h) = e(g^{f(\tau) - v}, h) = e(g, h)^{f(\tau) - v}
$$

右辺:

$$
e(\pi, h^\tau \cdot h^{-z}) = e(g^{q(\tau)}, h^{\tau - z}) = e(g, h)^{q(\tau)(\tau - z)}
$$

等式条件: $e(g, h)^{f(\tau) - v} = e(g, h)^{q(\tau)(\tau - z)}$、つまり:

$$
f(\tau) - v = q(\tau)(\tau - z)
$$

これは多項式恒等式 $f(X) - v = (X - z) q(X)$ を $X = \tau$ で評価したもの。正しい $q$ なら成立。◯

### 4.4 なぜ偽の $\pi$ を作れないか

もし Prover が偽の $v' \neq f(z)$ を主張し、偽の $\pi'$ を作ろうとする。このとき:

$$
f(X) - v' = (X - z) q'(X) + r, \quad r = f(z) - v' \neq 0
$$

$\pi' = g^{q'(\tau)}$ を作ると:

$$
e(C_f \cdot g^{-v'}, h) = e(g, h)^{f(\tau) - v'} = e(g, h)^{q'(\tau)(\tau - z) + r}
$$

$$
e(\pi', h^{\tau - z}) = e(g, h)^{q'(\tau)(\tau - z)}
$$

差分 $e(g, h)^r$ が残り、**等式が成立しない**。$r$ を消すには Prover が $\tau = z$ となる幸運を引く必要があり、$|\mathbb{F}| \approx 2^{256}$ で無視可能確率。

## 5. 安全性 (Hiding, Binding)

### 5.1 Binding

**Binding**: コミットメント $C_f$ に対し、$f$ とは違う $f' \neq f$ で同じ $C_f$ を作れない。

**KZG の binding**: **q-SBDH (q-Strong Bilinear Diffie-Hellman) 仮定**に帰着。

もし Prover が $C_f = g^{f(\tau)} = g^{f'(\tau)}$ と 2 つの多項式で表せれば、$(f - f')(\tau) = 0$。非ゼロ多項式 $f - f'$ が $\tau$ で 0 になる = $\tau$ はこの多項式の根 = $\tau$ を特定できる。これは q-SBDH に反する。

### 5.2 Hiding

**Hiding**: $C_f$ から $f$ の情報が漏れない。

**KZG の hiding**: **Discrete Log 仮定** のもとで computationally hiding。厳密には、KZG はそのままでは「部分的な情報」を漏らしうる（決定的にコミット）ので、ZK には**追加のマスキング**が必要（第6章）。

### 5.3 Knowledge Soundness

「受理される証明を作れる Prover は実際に $f$ を知る」。Knowledge of Exponent (KoE) や Algebraic Group Model (AGM) で正当化される。

## 6. ZK 化

### 6.1 素朴な KZG の情報漏れ

もし Prover が $f(z) = v$ を何度も同じ $z$ で評価させられると、$v$ から $f$ の情報が増える。これでは ZK ではない。

### 6.2 マスキング

KZG に ZK を加えるには、ランダムな**マスキング多項式** $r(X)$ を導入:

$$
\tilde f(X) = f(X) + r(X) \cdot Z_H(X)
$$

ここで $Z_H(X)$ は評価領域 $H$ 上でゼロになる多項式、$r(X)$ はランダムな低次多項式。

**評価点が $H$ 上なら** $r(X) Z_H(X) = 0$ なので元の値を保つ。**評価点が $H$ 外** なら $\tilde f$ の値はランダム化されて $f$ の情報を漏らさない。

### 6.3 実装上の細部

PLONK などでは、Witness ポリノミアル $w(X)$ に対して:

$$
\tilde w(X) = w(X) + (b_1 X + b_2) \cdot Z_H(X)
$$

のようにランダムな $b_1, b_2 \in \mathbb{F}$ を足す。これで 2 回以下の評価からは情報ゼロ（計算量的に）。

## 7. バッチ評価

### 7.1 複数多項式の同一点評価

Prover が $f_1, \ldots, f_k$ すべてについて $f_i(z) = v_i$ を示したい。

**素朴**: 各々に $\pi_i$ を作って送る → 総サイズ $O(k)$。

**バッチ化**: ランダムチャレンジ $\gamma$ を使い、線形結合:

$$
F(X) = f_1(X) + \gamma f_2(X) + \gamma^2 f_3(X) + \ldots
$$

$F(z) = v_1 + \gamma v_2 + \ldots$ を示せば、Schwartz-Zippel で各 $f_i(z) = v_i$ が確率 $\geq 1 - k/|\mathbb{F}|$ で成立。

証拠は 1 つの $\pi$ で済む → サイズ $O(1)$。

### 7.2 同一多項式の複数点評価

$f(z_1) = v_1, \ldots, f(z_k) = v_k$ の複数点評価。**補間多項式** $I(X)$（$I(z_i) = v_i$）を作り、$f - I$ が $\{z_1, \ldots, z_k\}$ で 0 → $Z_k(X) = \prod(X - z_i)$ で割り切れる。

商多項式 $q(X) = (f(X) - I(X))/Z_k(X)$ にコミットし、$k$ 点すべて一発で検証。

### 7.3 PLONK での使用

PLONK では、1つのプロトコル実行で多数の多項式を同じ点 $\zeta$ で評価するので、バッチ化が効く。ペアリングの回数を数回に抑えられる。

## 8. MPC Ceremony

### 8.1 なぜ Ceremony が必要か

$\tau$ を生成した人がそれを保持すると、**その人は偽証明を作れる**。したがって:

1. 誰が $\tau$ を生成したか不明にする
2. 終わったら $\tau$ を破棄する

### 8.2 Powers of Tau

複数人で協力して SRS を生成:

- 参加者 $i$ がランダムな $\tau_i$ を選ぶ
- 以前の SRS に $\tau_i$ を「掛ける」処理
- 最終 $\tau = \tau_1 \cdot \tau_2 \cdot \ldots \cdot \tau_n$

重要: **誰か 1 人が $\tau_i$ を破棄すれば $\tau$ は誰も知らない**。

### 8.3 有名な Ceremony

- **Zcash Powers of Tau**: 2018 年の大規模セレモニー
- **Ethereum KZG Ceremony**: 2023 年、14 万人以上が参加
- **Aztec, Filecoin**: 独自のセレモニー

多数の参加者がいれば、全員が結託しない限り $\tau$ は安全。

### 8.4 「毒」を破棄する儀式

初期の Zcash セレモニーでは、PC を物理的に破壊したり焼却したりした映像が有名（セキュリティ意識のデモンストレーション）。

## 9. 他の PCS との比較

### 9.1 FRI (Fast Reed-Solomon IOP of Proximity)

ハッシュベース。詳細 Article 23。

- **利点**: Transparent, PQ 耐性
- **欠点**: 証明サイズ 100KB〜、検証 log²(d)

### 9.2 IPA (Inner Product Argument)

Bulletproofs, Halo2 で使用。

- **利点**: Transparent
- **欠点**: 検証時間 $O(d)$、verifier が重い

### 9.3 Dory / Dark

Transparent かつペアリングベース。KZG と FRI の中間。

### 9.4 Brakedown

ハッシュベース、線形時間 Prover。
- **利点**: Prover が非常に速い
- **欠点**: 証明サイズが大きい（MB オーダー）

### 9.5 比較表

| PCS | 証明サイズ | Prover 時間 | Verify 時間 | Setup | PQ |
|---|---|---|---|---|---|
| **KZG** | $O(1)$ (48B) | $O(d \log d)$ | $O(1)$ | Trusted | ✗ |
| **FRI** | $O(\log^2 d)$ (100KB) | $O(d \log d)$ | $O(\log^2 d)$ | Transparent | ✓ |
| **IPA** | $O(\log d)$ (数KB) | $O(d)$ | $O(d)$ | Transparent | ✗ |
| **Dory** | $O(\log d)$ | $O(d)$ | $O(\log d)$ | Transparent | ✗ |
| **Brakedown** | $O(\sqrt d)$ | $O(d)$ | $O(\sqrt d)$ | Transparent | ✓ |

## 10. Q&A

### Q1: KZG の「$\tau$」は 1 人のものか？

**セレモニー参加者 1 人でも破棄すれば安全**。だから大規模セレモニーで参加者を増やす。

### Q2: KZG は Post-Quantum？

**いいえ**。ECDLP と q-SBDH に依存し、量子計算機で破れる。PQ 耐性が必要なら FRI を選ぶ。

### Q3: 実装で気をつける点は？

- **SRS の取り扱い**: 誰が生成したかの透明性
- **バッチ化**: ペアリング呼び出し回数を最小化
- **ドメイン**: 評価領域の選び方（単位根が良い）

### Q4: $\tau$ を知らない Prover が $g^{f(\tau)}$ をどう計算する？

SRS の $\{g^{\tau^i}\}$ を使って:

$$
g^{f(\tau)} = g^{\sum f_i \tau^i} = \prod (g^{\tau^i})^{f_i}
$$

これで $\tau$ を知らずに計算可能。計算は MSM。

### Q5: KZG のバッチ化はどれくらい効く？

PLONK では、1 証明で 10 以上の多項式を評価するが、最終的にペアリングは 2 回だけで済む。$O(k)$ ペアリングが $O(1)$ になる。

### Q6: SRS のサイズが大きいと何が困る？

- 配布が大変（数十〜数百MB）
- 検証側で保持する必要
- SRS から自分の $\mathbb{G}_1$ 要素だけ抽出する手法もある

## 11. まとめ

### 本記事の要点

1. **PCS** は多項式を固定サイズにコミットし、点評価を短く証明する仕組み
2. **KZG** はペアリングベースの代表 PCS、証明 48B・検証 $O(1)$
3. **Setup**: $g^{\tau^i}$ の列。$\tau$ は絶対秘密
4. **Commit**: $C_f = g^{f(\tau)}$
5. **Open**: $\pi = g^{q(\tau)}$、$q(X) = (f(X) - v)/(X - z)$
6. **Verify**: ペアリング等式 $e(C_f - v g, h) = e(\pi, h^\tau - z h)$
7. **ZK 化**: マスキング多項式 $r(X) Z_H(X)$ を加える
8. **バッチ評価**: 線形結合で複数評価を 1 つに圧縮
9. **Ceremony**: 多人数で $\tau$ を生成・破棄

### 次の記事（Article 14）へ

次の記事は **Sumcheck プロトコル** と **多重線形拡張 (Multilinear Extension, MLE)**。これらは多変数 Poly-IOP (Spartan, HyperPlonk) の主役で、**巨大な総和を対数回の対話で検証する**テクニック。

### 3行サマリ

- **KZG = 多項式を 48 バイトに封じ、1 点評価を 2 ペアリングで検証**
- **Trusted setup 必須、毒 $\tau$ を破棄する MPC セレモニー**
- **PLONK, Groth16, Marlin の心臓部**、他の PCS (FRI, IPA) と選択可能

---

## 参考文献

- Aniket Kate, Gregory Zaverucha, Ian Goldberg. *Constant-Size Commitments to Polynomials and Their Applications.* ASIACRYPT 2010.
- Alin Tomescu. *KZG Polynomial Commitments.* MIT course notes, 2020.
- Ariel Gabizon et al. *PLONK.* ePrint 2019/953.
- Ethereum Foundation. *KZG Ceremony Specification.* 2023.
- ZKP MOOC Lecture 6 (UC Berkeley, 2023).
