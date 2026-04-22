---
title: "PLONK — Universal Setup の革命"
emoji: "🎺"
type: "tech"
topics:
  - zkp
  - snark
  - plonk
  - permutation
  - kzg
published: false
---

**日付**: 2026年4月22日
**学習内容**: **PLONK** (Permutations over Lagrange-bases for Oecumenical Non-interactive Arguments of Knowledge, 2019) は、Groth16 の最大の弱点「回路ごとの Trusted Setup」を解消した SNARK。**1 度のセレモニーで複数の回路を証明できる Universal Setup** が最大の功績。本記事では **(1) PLONK の設計哲学**、**(2) 算術化の構造（gate constraints + wiring constraints）**、**(3) ラグランジュ基底での多項式表現**、**(4) ゲート制約の多項式化**、**(5) 配線制約: 置換引数 (permutation argument)**、**(6) grand product polynomial**、**(7) 全プロトコルの流れ**、**(8) Custom Gates と Plonkish** を追う。Groth16 の次を知るための必修記事。

## 0. 本記事の位置づけ

Article 15 で Groth16 を学んだ。その最大の問題は「**新しい回路ごとに MPC Ceremony が必要**」。これでは研究開発で回路を変えるたびに何十人〜何百人を招集する必要があり、実用で致命的。

PLONK の革新:

> **回路 $C_1$ 用にセレモニーした CRS で、後から回路 $C_2, C_3, \ldots$ も証明できる（回路サイズの上限以内で）**

これによりプロジェクト運用が劇的に楽になった。現代 SNARK (Aztec, Mina, zkSync, Scroll, Polygon zkEVM) はすべて PLONK 系。

構成:

- **第1章**: PLONK の目的
- **第2章**: 算術化: gate + wiring
- **第3章**: ラグランジュ基底と評価領域
- **第4章**: Gate constraints の多項式化
- **第5章**: Wiring constraints と置換引数
- **第6章**: Grand product polynomial
- **第7章**: 全プロトコル
- **第8章**: Custom Gates / Plonkish
- **第9章**: Q&A とまとめ

## 1. PLONK の目的

### 1.1 Universal Setup

**従来の Groth16**: 回路 $C$ ごとに CRS を生成。CRS は $C$ の多項式 $a_j, b_j, c_j$ に依存。

**PLONK**: CRS は「$\{g^{\tau^i}\}_{i=0}^{d}$」だけ。これは回路によらず共通。回路依存の部分は「事前処理 (preprocessing)」として公開され、Verifier が使う。

これで 1 度の ceremony で:

- 回路 $C_1, C_2, \ldots, C_k$ すべてに対応
- 回路サイズが $d$ 以下なら自由

### 1.2 トレードオフ

- **証明サイズ**: Groth16 (3 点) > PLONK (9 点くらい)
- **Prover 時間**: 同程度
- **Verifier 時間**: 同程度
- **Setup**: PLONK の方が遥かに柔軟

実用では、universal setup の価値が証明サイズの増加を上回る。

### 1.3 Plonkish への発展

PLONK の仕組みを拡張し、custom gate や lookup を追加したのが **Plonkish**。Halo2, Plonky2, Scroll などが採用。

## 2. 算術化: gate + wiring

### 2.1 算術回路の別の見方

Article 10 で「算術回路 = 有向非巡回グラフ」と見た。PLONK ではもう少し違う視点:

- **Gate**: 1 つの加算 or 乗算
- **Wire**: ゲートの入力・出力ライン

回路は**ゲートのリスト**と**ワイヤの接続関係**で完全に表現。

### 2.2 PLONK の基本ゲート形式

PLONK では、各ゲートを次の汎用制約で記述:

$$
q_L(X) a(X) + q_R(X) b(X) + q_M(X) a(X) b(X) + q_O(X) c(X) + q_C(X) = 0
$$

- $a, b, c$: 左入力、右入力、出力ワイヤ
- $q_L, q_R, q_M, q_O, q_C$: **セレクタ多項式**、ゲートの種類を決める

### 2.3 ゲート種別とセレクタの対応

| ゲート | $q_L$ | $q_R$ | $q_M$ | $q_O$ | $q_C$ |
|---|---|---|---|---|---|
| 加算 $a + b = c$ | 1 | 1 | 0 | -1 | 0 |
| 乗算 $a \cdot b = c$ | 0 | 0 | 1 | -1 | 0 |
| 定数 $c = k$ | 0 | 0 | 0 | -1 | $k$ |

各ゲートでセレクタが切り替わる = 回路全体で 1 つの制約式。

### 2.4 Wiring constraint (配線制約)

回路には「**あるゲートの出力が別のゲートの入力**」という接続関係がある。これを**置換 $\sigma$** で表す:

$$
\sigma : \{\text{wire positions}\} \to \{\text{wire positions}\}
$$

たとえば「ゲート 3 の出力 $c_3$ が ゲート 5 の左入力 $a_5$」なら $\sigma(c_3) = a_5$。

**Wiring constraint**: $\sigma(i) = j$ なら、対応する2つのワイヤは同じ値。

## 3. ラグランジュ基底と評価領域

### 3.1 評価領域 $H$

**$m$ 個のゲート**を持つ回路で、評価領域として:

$$
H = \{1, \omega, \omega^2, \ldots, \omega^{m-1}\}
$$

を使う。$\omega$ は 1 の $m$ 乗根。

### 3.2 ラグランジュ基底多項式

$H$ 上で $L_i(\omega^i) = 1$、$L_i(\omega^j) = 0$ ($i \neq j$) となる多項式:

$$
L_i(X) = \frac{X^m - 1}{m \cdot \omega^{-i} (X - \omega^i)}
$$

### 3.3 ワイヤの多項式表現

各ゲート $i$ の左入力を $a_i = a(\omega^i)$ と見る。全ゲートを通した左入力多項式:

$$
a(X) = \sum_{i=0}^{m-1} a_i \cdot L_i(X)
$$

**次数 $m - 1$**。同様に $b(X), c(X)$、そしてセレクタ $q_L(X), q_R(X), \ldots$。

### 3.4 消滅多項式

$Z_H(X) = \prod_{i=0}^{m-1}(X - \omega^i) = X^m - 1$。$H$ 上でゼロになる、次数 $m$ の多項式。

## 4. Gate constraints の多項式化

### 4.1 ゲート制約の多項式等式

各ゲートで:

$$
q_L(\omega^i) a(\omega^i) + q_R(\omega^i) b(\omega^i) + q_M(\omega^i) a(\omega^i) b(\omega^i) + q_O(\omega^i) c(\omega^i) + q_C(\omega^i) = 0
$$

多項式レベル:

$$
F_{\text{gate}}(X) := q_L(X) a(X) + q_R(X) b(X) + q_M(X) a(X) b(X) + q_O(X) c(X) + q_C(X)
$$

**$F_{\text{gate}}(X)$ が $H$ 上でゼロ** ⟺ すべてのゲート制約が成立。

すなわち $F_{\text{gate}}(X) = Z_H(X) \cdot T(X)$ となる $T(X)$ が存在する。

### 4.2 次数の管理

- $q, a, b$: 次数 $m - 1$
- $q_M \cdot a \cdot b$: 次数 $3(m-1)$
- $F_{\text{gate}}$: 次数 $3(m-1)$
- $Z_H$: 次数 $m$
- $T$: 次数 $3(m-1) - m = 2m - 3$

## 5. Wiring constraints と置換引数

### 5.1 配線制約の難しさ

ゲート $i$ の出力が ゲート $j$ の左入力のとき、$c(\omega^i) = a(\omega^j)$ を保証する必要がある。これは**ゲート制約ではなく、異なる多項式の間の等式**。素朴に制約を並べると膨大になる。

PLONK の賢い解決: **置換引数 (permutation argument)**。

### 5.2 置換引数の発想

3 つのワイヤ多項式 $a(X), b(X), c(X)$ を「**3 つの系列**」と見て、それぞれに**識別子** (position) を割り当てる:

- $a$ のポジション: $1, 2, \ldots, m$
- $b$ のポジション: $m+1, m+2, \ldots, 2m$
- $c$ のポジション: $2m+1, 2m+2, \ldots, 3m$

合計 $3m$ ポジション。

置換 $\sigma$ はこれらのポジション上の置換。

### 5.3 Grand Product 等式

置換 $\sigma$ が **wire identity** ($\sigma(i) = j \Rightarrow \text{wire}_i = \text{wire}_j$) を保つことを、次の **grand product** で検証する。

ランダムチャレンジ $\beta, \gamma$ を使い:

$$
\prod_{i=0}^{3m-1} \frac{w_i + \beta \cdot \text{id}(i) + \gamma}{w_i + \beta \cdot \sigma(i) + \gamma} = 1
$$

ここで $\text{id}(i) = i$、$w_i$ は位置 $i$ のワイヤ値。

### 5.4 Grand Product の直感

置換 $\sigma$ が正しい wire identity を表していれば、分子分母は同じ要素の並び替えなので積は 1。さもなければ Schwartz-Zippel で確率 $O(m/|\mathbb{F}|)$ を除き、1 にならない。

### 5.5 多項式形式

Grand product を段階的に計算する多項式 $Z(X)$ を定義:

$$
Z(\omega^0) = 1, \quad Z(\omega^{i+1}) = Z(\omega^i) \cdot \frac{(a_i + \beta \text{id}_a(i) + \gamma)(b_i + \beta \text{id}_b(i) + \gamma)(c_i + \beta \text{id}_c(i) + \gamma)}{(a_i + \beta \sigma_a(i) + \gamma)(b_i + \beta \sigma_b(i) + \gamma)(c_i + \beta \sigma_c(i) + \gamma)}
$$

**閉条件**: $Z(\omega^m) = 1$（= $Z(\omega^0)$ に一周戻る）。

### 5.6 Wiring 制約の多項式等式

$Z(X)$ の再帰関係を多項式等式として書くと:

$$
Z(X \omega) \cdot [(\sigma_a \text{ 項}) \cdots] = Z(X) \cdot [(\text{id}_a \text{ 項}) \cdots]
$$

これが $H$ 上で成立する必要。**Gate と同じく消滅多項式で割り切れ**を示す。

## 6. Grand product polynomial の詳細

### 6.1 なぜ積が 1 なのか

**補題**: 数列 $\{a_i\}_{i=1}^n$ と $\{b_i\}_{i=1}^n$ が並び替えなら、ランダム $\alpha$ に対して $\prod_i (\alpha - a_i) = \prod_i (\alpha - b_i)$（多項式として）。

$\beta, \gamma$ を使う形式はこれの変種で、witness に依存してランダム化されている。

### 6.2 健全性の確率

もし $\sigma$ が wire identity を保たない（偽の witness）なら、ランダムチャレンジ $(\beta, \gamma)$ に対して grand product が 1 になる確率 $\leq 3m/|\mathbb{F}|$（Schwartz-Zippel）。

### 6.3 計算コスト

Prover は $Z(X)$ を $O(m)$ 時間で計算（各ステップ定数時間）。その後コミットメント。

## 7. 全プロトコル

### 7.1 Setup (Universal)

$\tau$ をランダムに選び:

$$
\text{CRS} = \{g^{\tau^i}\}_{i=0}^{d}
$$

回路の具体情報（$q_L, q_R, q_M, q_O, q_C, \sigma_a, \sigma_b, \sigma_c$）は**事前処理**で公開（Verifier が使う）。

### 7.2 Prover の手順

1. Witness から $a(X), b(X), c(X)$ を構成
2. Grand product $Z(X)$ を計算
3. 消滅多項式 $Z_H$ での除算で $T(X)$ を得る（gate + wiring の全制約をまとめた quotient）
4. すべての多項式を KZG コミット
5. Fiat-Shamir で challenge $\zeta$ を生成
6. 各多項式を $\zeta$ で評価し、KZG opening を計算
7. 証明 $\pi = (\text{commits}, \text{evals}, \text{opening})$ を送信

### 7.3 Verifier の手順

1. 各コミットメントの正当性確認
2. 評価値が整合的かどうか、ペアリング等式で検証
3. Gate constraint: $F_{\text{gate}}(\zeta) = Z_H(\zeta) T(\zeta)$
4. Wiring constraint: Grand product の等式
5. すべて成立なら受理

### 7.4 証明サイズ

- コミットメント: $a, b, c, Z, T$ の 5 点（大きな $T$ は分割）
- 評価値: 数個のスカラー
- 合計: 約 9 点 ($\approx 500$ bytes)

### 7.5 検証時間

ペアリングは KZG opening のバッチ検証で **2 回**。これは Groth16 より若干多いが、まだ数 ms。

## 8. Custom Gates と Plonkish

### 8.1 Custom Gate の導入

PLONK の基本ゲート形式は「加算・乗算」だけ。もっと複雑な計算（ビット分解、ハッシュの S-box）を**1 ゲートで**表したい。

Plonkish:

$$
q_L a + q_R b + q_M a b + q_O c + q_C + q_{\text{custom}} \cdot \text{custom\_gate}(a, b, c, \ldots) = 0
$$

カスタムゲートは、たとえば:

$$
\text{poseidon\_sbox}(a, c) = a^5 - c
$$

これで S-box を 1 ゲートで済ませる。Poseidon ハッシュの制約数が激減。

### 8.2 Lookup Argument

「$a$ がテーブル $T$ にある」を示す **lookup** は、Plonkish の大きな武器。Article 18 で詳述。

### 8.3 Higher-degree gates

Plonkish では**高次ゲート**も許容:

$$
q \cdot (a^d + b^d + c^d + \ldots) = 0
$$

次数 $d$ が大きいと Prover コストが上がるが、特定用途で劇的な効率化。

### 8.4 Plonkish を採用する SNARK

- **Halo2** (Zcash Orchard, Scroll)
- **Plonky2** (Polygon Zero)
- **Plonky3** (次世代、field-agnostic)

どれも「PLONK のアイデアを拡張して、回路記述を自由にする」方向。

## 9. Q&A

### Q1: なぜ PLONK の名前は「Permutation」？

Wiring constraint を置換引数 (permutation argument) で扱うのが核心的アイデアだから。

### Q2: Grand Product の $Z(X)$ と消滅多項式の $Z(X)$、混同しない？

文献によって表記が違う。本記事では:
- **消滅多項式**: $Z_H(X)$
- **Grand product 多項式**: $Z(X)$

区別して書くのが良い。

### Q3: PLONK の証明は Groth16 より大きい？

**はい**、約 2〜3 倍（Groth16: 3 点、PLONK: 9 点）。ただし**検証時間は同程度**、Universal Setup の価値で十分元が取れる。

### Q4: PLONK は FFT が必須？

**はい**。ラグランジュ基底での多項式操作に FFT を多用。評価領域 $|H|$ の FFT が必須 = 対応する素数 $p$ が必要。

### Q5: 既存の Groth16 回路を PLONK に移行できる？

R1CS → Plonkish の翻訳は可能。ただし**最適化し直した方が効率が良い**（custom gate を活かすなど）。実用では、新規プロジェクトは PLONK 系で書く。

### Q6: PLONK → Halo2 への進化は？

Halo2 は:
- Plonkish（PLONK + custom gate）
- **IPA (Inner Product Argument)** を使用（KZG の代わり）
- **Transparent Setup**（Trusted Setup 不要）
- **再帰 SNARK** (Article 19)

PLONK の良さ（universal）を保ちつつ、setup を透明化。

## 10. まとめ

### 本記事の要点

1. **PLONK**: Universal Setup を実現した革命的 SNARK
2. **ゲート形式**: $q_L a + q_R b + q_M ab + q_O c + q_C = 0$
3. **Wiring constraint**: 置換引数 (grand product) で表現
4. ラグランジュ基底で多項式表現、$H = \{\omega^i\}$ 上で評価
5. KZG コミットメントと Fiat-Shamir で非対話化
6. 証明サイズ約 9 点、検証約 2 ペアリング
7. **Plonkish**: custom gate, lookup で拡張、Halo2/Plonky2 で採用

### 次の記事（Article 17）へ

次の記事は **Fiat-Shamir 変換**。対話型プロトコルを「Verifier の乱数をハッシュ関数に置き換える」ことで非対話化する仕組み。PLONK も Sumcheck もすべてこの変換に依存。

### 3行サマリ

- **PLONK = Universal Setup + 汎用ゲート形式 + 置換引数**
- **1 度の ceremony で複数回路**、実用の最大の便益
- **Plonkish への拡張**: Halo2, Plonky2 で現代 SNARK の基盤

---

## 参考文献

- Ariel Gabizon, Zachary J. Williamson, Oana Ciobotaru. *PLONK: Permutations over Lagrange-bases for Oecumenical Non-interactive Arguments of Knowledge.* ePrint 2019/953.
- Vitalik Buterin. *Understanding PLONK.* Blog, 2019.
- ZKP MOOC Lecture 5 (UC Berkeley, 2023).
- ZKTokyo Week 3 資料.
- Electric Coin Company. *Halo2 Book.* 2023.
