# Recursive SNARKs（再帰的SNARK）完全解説

> **出典**: ZKP MOOC Lecture 10 — Dan Boneh, Shafi Goldwasser, Dawn Song, Justin Thaler, Yupeng Zhang  
> (Stanford University / UC Berkeley / Georgetown University / Texas A&M University)

---

## 目次

1. [SNARKとは何か？（前提知識の整理）](#1-snarkとは何か前提知識の整理)
2. [2段階SNARK再帰とは](#2-2段階snark再帰とは)
3. [再帰が安全である理由（健全性の証明）](#3-再帰が安全である理由健全性の証明)
4. [ランダムオラクルの問題](#4-ランダムオラクルの問題)
5. [応用1：証明の圧縮（Proof Compression）](#5-応用1証明の圧縮proof-compression)
6. [応用2：ストリーミング証明生成（zk-Rollup）](#6-応用2ストリーミング証明生成zk-rollup)
7. [応用3：レイヤー3 zk-Rollup](#7-応用3レイヤー3-zk-rollup)
8. [応用4：IVC（インクリメンタル検証可能計算）](#8-応用4ivcインクリメンタル検証可能計算)
9. [応用5：ZK証明者マーケットプレイス](#9-応用5zk証明者マーケットプレイス)
10. [再帰をサポートする楕円曲線の選び方](#10-再帰をサポートする楕円曲線の選び方)
11. [Nova：折りたたみ（Folding）による効率的な再帰](#11-nova折りたたみfolding-による効率的な再帰)
12. [まとめと発展](#12-まとめと発展)

---

## 1. SNARKとは何か？（前提知識の整理）

### SNARKのざっくり説明

**SNARK**（Succinct Non-interactive ARgument of Knowledge）とは、ある計算が正しく行われたことを「証拠（proof）」で証明する技術です。

ポイントは3つです：
- **Succinct（簡潔）**：証明のサイズが小さい（計算量に比べてずっと短い）
- **Non-interactive（非対話型）**：証明者と検証者が何度もやり取りしなくてよい
- **Argument of Knowledge（知識の証明）**：ある秘密の情報（witness）を本当に知っていることを証明できる

### 数学的定義（前処理型SNARK）

前処理型SNARKは3つのアルゴリズムの組 $(S, P, V)$ で構成されます：

$$S(C) \rightarrow \text{公開パラメータ} (pp, vp)$$

$$P(pp, \boldsymbol{x}, \boldsymbol{w}) \rightarrow \text{証明} \pi$$

$$V(vp, \boldsymbol{x}, \pi) \rightarrow \text{accept または reject}$$

- $C$：回路（計算内容を表す）
- $\boldsymbol{x}$：公開入力（検証者も知っている値）
- $\boldsymbol{w}$：秘密の証人（witnessウィットネス、証明者だけが知っている値）
- $\pi$：証明
- $pp$：証明者用パラメータ
- $vp$：検証者用パラメータ

### 知識健全性（Knowledge Soundness）

SNARKが「本物の証明」であることを保証する性質です。

$$\Pr[C(y, w) = 0 : w \leftarrow E(pp, y)] \geq \Pr[V(vp, y, A(pp, y)) = \text{yes}] - \varepsilon$$

- 左辺：抽出器 $E$ が有効な秘密情報 $w$ を取り出せる確率
- 右辺：悪意ある証明者 $A$ が検証者を騙せる確率
- $\varepsilon$：無視できるほど小さな誤差（ネグリジブル）

**平たく言うと**：正しい証明を出せる人は、必ずその証明の根拠となる秘密情報を持っているはずだ、ということです。

### 代表的なSNARK

| 方式 | 証明サイズ | 証明者の速度 |
|------|-----------|------------|
| **Groth16 / Plonk-KZG** | 短い（~数百B） | $O(n \log n)$（やや遅い） |
| **FRI系（Breakdown, Orion等）** | 長い（~100KB） | 速い |

---

## 2. 2段階SNARK再帰とは

### 基本アイデア

「SNARKの検証結果を、さらにSNARKで証明する」というのが再帰的SNARKの核心です。

```
公開: x           公開: vp, x
秘密: w            秘密: π
   ↓                  ↓
SNARK証明者P  →π→  SNARK証明者P'  →π'→  最終的な検証者
(S, P, V)           (S', P', V')

P は「C(x,w)=0 を満たすwを知っている」を証明
P' は「V(vp,x,π)=yes となるπを知っている」を証明
```

### なぜこれが便利なのか？

最終的な検証者が確認するのは $\pi'$ だけでよく、内側の証明 $\pi$ の詳細を見る必要がありません。これにより：

1. 証明の**圧縮**（大きい証明→小さい証明）
2. 証明の**連鎖**（複数の証明を一本化）

が可能になります。

---

## 3. 再帰が安全である理由（健全性の証明）

### 2段階再帰の健全性の証明スケッチ

**設定**：

- 回路 $C: \mathbb{F}^n \times \mathbb{F}^m \rightarrow \mathbb{F}$ を固定
- $(pp, vp) \leftarrow S(C)$ とする
- 補助回路 $C'$ を定義：

$$C'((vp, x), \pi) := [V(vp, x, \pi) == \text{yes}]$$

**証明の流れ**：

**Step 1**：$(S', P', V')$ は $C'$ について知識健全であるため、悪意ある証明者 $A$ から抽出器 $E'$ が $\pi$ を取り出せる：

$$V(vp, x, \pi) = \text{yes}$$

**Step 2**：$(S, P, V)$ は $C$ について知識健全であるため、$E'$ を「賢い証明者」として使い、抽出器 $E$ が $w$ を取り出せる：

$$C(x, w) = 0$$

### 成功確率の連鎖

$$\Pr[C(x, w) = 0] \geq \Pr[\pi' \leftarrow A(pp', x) \text{ は有効}] - \varepsilon' - \varepsilon$$

### ⚠️ 重大な注意点：再帰の深さ制限

抽出器の実行時間は再帰の深さとともに指数的に増加します：

$$\text{time}(E^{(n)}) = 2^n \times \text{time}(A)$$

これは多項式時間ではありません。そのため、**再帰の深さは $\log(\lambda)$（セキュリティパラメータの対数）を超えてはならない**という制限があります。

---

## 4. ランダムオラクルの問題

### Fiat-Shamir変換と再帰

Fiat-Shamir変換を使うと、対話型の証明を非対話型（SNARK）に変換できます。しかし、内部の検証者 $V$ がランダムオラクル（RO）を使う場合、再帰の中でそれをどう扱うかが問題になります。

**解決策**：
- 検証者のROを具体的なハッシュ関数 $H$（例：SHA-256, Poseidon）で置き換える
- 得られた $(S_H, P_H, V_H)$ が安全であると**仮定する**
- この仮定のもとで再帰を実行

**欠点**：セキュリティ証明に「やや強い仮定」が必要になります。

---

## 5. 応用1：証明の圧縮（Proof Compression）

### 問題設定

- **FRI系SNARK**：証明者は速いが、証明のサイズが大きい（例：100KB）
- **Groth16/Plonk-KZG**：証明者は遅いが、証明サイズは小さい（例：1KB）

### 解決策：2段階再帰

```
公開: x          公開: vp, x
秘密: w           秘密: π(100KB)
   ↓                   ↓
FRI-SNARK(P) →π(100KB)→ Groth16(P') →π'(1KB)→ 検証者

全体として：証明者は高速（FRI）で、最終証明は短い（1KB）
```

**ポイント**：
- 1段目（FRI）：高速な証明生成 → 大きい証明
- 2段目（Groth16）：遅いが、小さい証明に圧縮

最終的に「高速かつ証明が短い」という両立を実現します。

---

## 6. 応用2：ストリーミング証明生成（zk-Rollup）

### zk-Rollupの基本

**zk-Rollup**とは、Ethereumのスケーリング技術で、多数のトランザクション（Tx）を1つの証明にまとめてLayer-1（L1）に提出するものです。

### 問題：従来の遅さ

```
Tx1〜Tx10 | Tx11〜Tx20 | ... | Tx91〜Tx100

     → 100件全て揃ってから証明生成開始 → 長い遅延
```

全トランザクションが揃うまで証明を生成できないため、遅延が大きい。

### 解決策：ストリーミング証明生成

```
Tx1〜10 → 証明π₁生成
Tx11〜20 → 証明π₂生成
...
Tx91〜100 → 証明π₁₀生成
               ↓
         π₁,...,π₁₀ を1つに合成 → 最終証明π
```

再帰的SNARKを使うことで、トランザクションが来るたびに部分的な証明を生成し、最後にそれらをまとめることができます。遅延が大幅に短縮されます。

---

## 7. 応用3：レイヤー3 zk-Rollup

### Rollupのレイヤー構造

```
L3（ゲーム等） ←→ L2（Gaming Co.） ←→ L1（Ethereum本体）
```

- **L1**（Layer-1）：Ethereumブロックチェーン本体
- **L2**（Layer-2）：L1上のRollupコントラクト（資産とMerkle状態根を保持）
- **L3**（Layer-3）：L2上でさらに動くRollup（カスタム実行エンジンを使いたい場合）

### なぜL3が必要か？

ゲーム会社を例に考えます：
- L2はEVM（Ethereum Virtual Machine）を使う制約がある
- ゲーム専用の高速な実行エンジンを使いたい
- L1→L2よりも高速な決済レートが欲しい

### L3の動作フロー

1. **1秒ごと**：L3コーディネーター → L2コーディネーターに新しいL3状態根と遷移証明 $\pi_3$ を送信
2. **L2 Rollupコントラクト**：証明を確認し、更新されたL3状態根を記録
3. **1分ごと**：L2コーディネーターは60個の $\pi_3$ をまとめた再帰証明 $\pi_2$ を生成
4. **L1コントラクト**：$\pi_2$ を確認し、L2状態根を更新

ここで再帰的SNARKが活躍します。60個の証明を1つにまとめる操作がまさに再帰です。

---

## 8. 応用4：IVC（インクリメンタル検証可能計算）

### IVCとは？ [Valiant 2008]

「関数 $F$ を繰り返し適用する長い計算」に対して、途中の各ステップで「ここまでの計算が正しい」という証明を段階的に積み上げていく技術です。

### 計算の図

$$s_0 \xrightarrow{F(\cdot, \omega_1)} s_1 \xrightarrow{F(\cdot, \omega_2)} s_2 \xrightarrow{\cdots} s_{n-1} \xrightarrow{F(\cdot, \omega_n)} s_n$$

- $s_0$：初期状態（公開）
- $\omega_i$：各ステップの秘密の入力（witness）
- $s_n$：最終出力（公開）

**目標**：「$\omega_1, \ldots, \omega_n$ を知っており、最終出力 $s_n$ が正しい」という簡潔な証明を生成する。  
検証者は $F$、$n$、$s_0$、$s_n$ を知っている。

### IVCの構築アイデア

各ステップ $i$ において、証明者は $s_i$ と証明 $\pi_i$ を出力します。$\pi_i$ は次のことを証明します：

「$s_{i-1}$、$\omega_i$、$\pi_{i-1}$ を知っており、次の2条件を満たす：

$$F(s_{i-1}, \omega_i) = s_i \quad \text{かつ} \quad V(vp, (i{-}1, s_0, s_{i-1}), \pi_{i-1}) = \text{yes}$$

最終証明 $\pi_n$ だけを見れば、「$n$ ステップの計算全体が正しい」ことがわかります。

### IVCの応用例

#### 1. マイクロプロセッサのステップごとの証明

$F$：CPUの1命令実行（Risc-V、EVM等）  
→ 巨大な計算を細かいステップに分割し、各ステップで少ないメモリで証明生成できる

#### 2. ブロックチェーン全体の状態証明（Minaブロックチェーン）

$$s_0 = \text{創世ブロックの状態}, \quad \omega_i = \text{第}i\text{ブロックのトランザクション群}$$

1つの再帰証明を確認するだけで、ブロックチェーン全体の歴史が正しいことを検証できます。  
**Mina Protocol**はこの技術を活用しており、ブロックチェーン全体のサイズが常に一定（~22KB）に保たれています。

#### 3. 検証可能遅延関数（VDF）

$$s_0 \xrightarrow{H} s_1 \xrightarrow{H} s_2 \xrightarrow{\cdots} \xrightarrow{H} s_n$$

「$s_n = H^n(s_0)$（ハッシュを $n$ 回適用）」という証明を簡潔に提供できます。乱数生成やタイムロック暗号等に使われます。

---

## 9. 応用5：ZK証明者マーケットプレイス

### 発想

ZK証明の生成はGPUを大量に必要とする重い計算です。これを**分散化**し、誰でも証明生成に参加して報酬を得られる仕組みです。

### 仕組み

```
tx1, tx2 → [証明者1] → π₁ ─┐
                             ├→ [証明者3] → π（最終証明）
tx3, tx4 → [証明者2] → π₂ ─┘

           ↑
     マーケットが証明者を選定し報酬を分配
```

再帰的SNARKにより、証明者1と証明者2の部分証明を証明者3が1つの証明に統合できます。証明者間での協力が可能になります。

---

## 10. 再帰をサポートする楕円曲線の選び方

### なぜ楕円曲線が問題になるのか？

SNARKの証明（特にKZG多項式コミットメント）は**楕円曲線上の群 $\mathbb{G}$** を使います。

重要な関係：

- 群 $\mathbb{G}$ の**位数**が $p$（証明したい計算の体 $\mathbb{F}_p$ のサイズ）
- 群 $\mathbb{G}$ は体 $\mathbb{F}_q$ 上で定義される（$p \neq q$ が一般的）

つまり、**証明者は $\mathbb{F}_p$ 上の算術を使う**が、**検証者は $\mathbb{F}_q$ 上の群演算が必要**という「体のミスマッチ」が生じます。

### 解決策の選択肢

#### Option 1：体のエミュレーション（Field Emulation）

$\mathbb{F}_p$ 上の回路で $\mathbb{F}_q$ の算術を実装する（例：KZGの評価証明回路）。

**問題点**：検証者回路が巨大になり、証明者の速度が著しく低下する。

#### Option 2：都合のよい群の探索

「位数 $p$、体 $\mathbb{F}_p$ 上で定義される群 $\mathbb{G}$」を探す。

**問題点**：そのような群では離散対数問題が容易（安全でない）ため、実用的なものは存在しない。

#### Option 3（実用的）：群のチェーン／サイクル

**チェーン**：

$$\mathbb{G}_1 \subseteq \mathbb{F}_q, \quad |\mathbb{G}_1| = p$$

$$\mathbb{G}_2 \subseteq \mathbb{F}_r, \quad |\mathbb{G}_2| = q$$

→ $\mathbb{G}_1$ の検証者が必要とする $\mathbb{F}_q$ の算術を、$\mathbb{G}_2$ の証明者がサポートする。  
長いチェーンで多段階の再帰が可能。

**サイクル（2-cycle）[BCTV'14]**：

$$|\mathbb{G}_1| = p, \quad \mathbb{G}_1 \subseteq \mathbb{F}_q$$

$$|\mathbb{G}_2| = q, \quad \mathbb{G}_2 \subseteq \mathbb{F}_p$$

2つの証明システムが互いの体をサポートし合うため、**無限に再帰できます**（2つの証明システムを交互に使う）。

### 3種類の2-サイクル

| 種類 | 説明 | 実用性 |
|------|------|--------|
| **タイプ1** | $\mathbb{G}_1$、$\mathbb{G}_2$ ともにペアリング群 | 非効率な構造しか知られていない |
| **タイプ2** | $\mathbb{G}_1$ はペアリング群、$\mathbb{G}_2$ は通常の群 | KZG + Bulletproofs の組み合わせ |
| **タイプ3** | どちらもペアリング群でない | **Pastaカーブ**（Pallas/Vesta）が有名 |

### Pastaカーブ（Halo2で使用）

楕円曲線 $E/\mathbb{Q}: y^2 = x^3 + 5$ を基に構成されます。

多くの素数 $q$ に対して、$p = |E(\mathbb{F}_q)|$ が素数ならば：

$$|E(\mathbb{F}_p)| = q \quad \text{かつ} \quad |E(\mathbb{F}_q)| = p$$

Pallas曲線（$y^2 = x^3 + 5$ over $\mathbb{F}_p$）とVesta曲線（$y^2 = x^3 + 5$ over $\mathbb{F}_q$）のペアが、Halo2プロトコルで採用されています。

---

## 11. Nova：折りたたみ（Folding）による効率的な再帰

### フル再帰の問題点

従来の再帰では、各ステップで「検証アルゴリズム $V(vk, x, \pi)$ 全体を回路として実行」する必要がありました。これは非常に高コストです：

- 多項式コミットメントの評価証明検証
- ペアリング演算（KZGの場合）

**Nova**（[Kothapalli, Setty, Tzialla 2021]）はこの問題を解決します。

### Novaの核心：折りたたみスキーム

「2つのR1CSインスタンスを1つに圧縮する」プロトコルです。

#### R1CS（Rank-1 Constraint System）とは

あらゆる回路は R1CS に変換できます。行列 $A, B, D \in \mathbb{F}_p^{u \times v}$ に対して：

$$Az \circ Bz = Dz$$

ここで $z = (x, w) \in \mathbb{F}_p^v$（$x$: 公開入力、$w$: 秘密の証人）。  
$\circ$ はアダマール積（成分ごとの積）：$(a_1, a_2) \circ (b_1, b_2) = (a_1 b_1, a_2 b_2)$。

#### 緩和R1CS（Relaxed R1CS）

通常の R1CS を少し緩めた形式：

$$Az \circ Bz = c(Dz) + E$$

- $c \in \mathbb{F}_p$：スカラー係数
- $E \in \mathbb{F}_p^u$：誤差ベクトル（error term）

通常の R1CS は $c = 1, E = 0$ の特別な場合です。

#### コミット済み緩和R1CS

$E$ が大きいため、直接持つのではなくコミットメントで置き換えます：

- **検証者**が持つ公開インスタンス：$(x, c, \text{com}_E)$
- **証明者**が持つ証人：$(z, E, r_E)$（ここで $\text{com}_E = \text{commit}(E, r_E)$）

#### 折りたたみの手順

インスタンス1：$(x_1, c_1, \text{com}_{E_1})$、証人 $(z_1, E_1, r_{E_1})$  
インスタンス2：$(x_2, c_2, \text{com}_{E_2})$、証人 $(z_2, E_2, r_{E_2})$

**Step 1**（証明者）：クロス項 $T$ を計算し、そのコミットメントを送る：

$$T \leftarrow (Az_2) \circ (Bz_1) + (Az_1) \circ (Bz_2) - c_1(Dz_2) - c_2(Dz_1)$$

$$\text{com}_T \leftarrow \text{commit}(T, r_T) \quad \text{（検証者に送信）}$$

**Step 2**（検証者）：ランダムな $r \leftarrow \mathbb{F}_p$ を選び、折りたたまれたインスタンスを計算：

$$x \leftarrow x_1 + r x_2$$

$$c \leftarrow c_1 + r c_2$$

$$\text{com}_E \leftarrow \text{com}_{E_1} + r \cdot \text{com}_T + r^2 \cdot \text{com}_{E_2}$$

（コミットメントの加法的準同型性を利用）

**Step 3**（証明者）：折りたたまれた証人を計算：

$$z \leftarrow z_1 + r z_2, \quad E \leftarrow E_1 + rT + r^2 E_2, \quad r_E \leftarrow r_{E_1} + r \cdot r_T + r^2 \cdot r_{E_2}$$

#### なぜ正しいのか？

$$Az \circ Bz = A(z_1 + rz_2) \circ B(z_1 + rz_2)$$

$$= (Az_1)(Bz_1) + r^2(Az_2)(Bz_2) + r\bigl[(Az_2)(Bz_1) + (Az_1)(Bz_2)\bigr]$$

$$= \bigl[c_1(Dz_1) + E_1\bigr] + r^2\bigl[c_2(Dz_2) + E_2\bigr] + rT$$

$$= (c_1 + rc_2)(Dz_1 + rDz_2) + E_1 + rT + r^2 E_2$$

$$= c(Dz) + E \quad \checkmark$$

折りたたんで得られた $(x, c, E, z)$ は緩和R1CSの有効な解になっています。

### 加法的準同型コミットメント

折りたたみスキームには、コミットメントが**加法的準同型**であることが必要です：

$$\text{commit}(m_1, r_1) + \text{commit}(m_2, r_2) = \text{commit}(m_1 + m_2, r_1 + r_2)$$

Pedersen コミットメントや格子ベースのコミットメントがこの性質を持ちます。  
検証者側の計算は「2つの群上の掛け算（スカラー倍）」のみで済みます：

$$\text{com}_E \leftarrow \text{com}_{E_1} + r \cdot \text{com}_T + r^2 \cdot \text{com}_{E_2}$$

### NovaによるIVCの実装

各ステップ $i$ の R1CS インスタンス：

$$x_i = (i, s_0, s_i, s_{i+1}, \omega_i)$$

チェックする内容：

$$s_{i+1} = F(s_i, \omega_i) \quad \text{（かつ $i=0$ なら $s_i = s_0$）}$$

**折りたたみを繰り返す**：

$$x_1, x_2 \xrightarrow{\text{fold}} x_{12} \xrightarrow{\text{fold with } x_3} x_{123} \xrightarrow{\text{fold with } x_4} \cdots \xrightarrow{\text{fold}} x_{1\cdots n}$$

**最後に**：残った1つのインスタンス $(x_{1\cdots n}, c, \text{com}_E)$ に対して通常の SNARK で証明を生成。

### 各ステップの証明者の仕事

各折りたたみステップで証明者がやること：

1. **関数 $F$ の評価**
2. **$\mathbb{G}$ 上の2回の乗算**（コミットメント計算）
3. **簡単なハッシュ計算**（Fiat-Shamir のため）

これは「SNARK 検証回路全体を実行する」より**はるかに高速**です。

### Fiat-Shamir化と追加の確認

非対話型にするためにFiat-Shamir変換を適用：

$$r \leftarrow H(x_{1\cdots i}, x_{i+1}, \text{com}_T, \ldots)$$

ただし、折りたたみが正しく行われたことも証明する必要があります。そのため、拡張されたR1CSプログラム $(A', B', D')$ が必要で、これは：

- インスタンス $x_i$ に対する証人が有効であることを検証
- $x_{1\cdots i+1}$ が $x_{1\cdots i}$ と $x_i$ の正しい折りたたみ結果であることを検証

これには $\mathbb{G}$ 上の2回の乗算が含まれるだけなので、非常に軽量です。

### Supernova（Novaの拡張）

- **Nova**：同じ関数 $F$ を繰り返し適用する場合に最適化
- **Supernova**：$F_1, F_2, \ldots, F_k$ と**異なる関数を混在**させてIVCができる
  - 各関数 $F_i$ に対して別々にNovaを適用

### Sangria（Plonk向けの折りたたみ）

- Novaは**R1CS（二次制約）** に対する折りたたみ
- **Sangria**は**Plonk arithmetization**に対応した折りたたみスキーム
- Plonkの算術を使って $F$ を表現し、効率的なIVCを実現

---

## 12. まとめと発展

### この講義で学んだこと

```
再帰的SNARK
├── 基本概念：SNARKの証明自体をSNARKで証明する
├── 安全性：2段の抽出器の連鎖で知識健全性を証明
│   └── 注意：再帰深さはlog(λ)まで（指数的コスト爆発）
├── 応用
│   ├── 1. 証明圧縮（FRI + Groth16の組み合わせ）
│   ├── 2. ストリーミング証明（zk-Rollupの遅延削減）
│   ├── 3. レイヤー3 Rollup（再帰的状態遷移証明）
│   ├── 4. IVC（Mina、VDF等）
│   └── 5. 証明者マーケットプレイス
├── 楕円曲線の選択
│   ├── 体のミスマッチ問題
│   ├── 群のチェーン
│   └── 群のサイクル（Pasta曲線 = Pallas + Vesta）
└── 折りたたみスキーム（Nova）
    ├── 緩和R1CS
    ├── コミット済み緩和R1CS
    ├── 折りたたみの具体的手順
    ├── 加法的準同型コミットメント
    └── 拡張：Supernova（複数関数）、Sangria（Plonk）
```

### 重要な論文・実装

| 名称 | 概要 |
|------|------|
| **Valiant 2008** | IVCの初の理論的構築 |
| **BCTV 2014** | 楕円曲線の2-サイクルによる実用的再帰 |
| **Halo2** | Pastaカーブを使った実用的な再帰SNARK |
| **Nova (2021)** | 折りたたみによる効率的IVC（eprint.iacr.org/2021/370） |
| **Supernova** | Novaの複数関数対応拡張（eprint.iacr.org/2022/1758） |
| **Sangria** | Plonk向け折りたたみスキーム |
| **Mina Protocol** | IVCを用いた定サイズブロックチェーン |

### DeInへの関連性

DeIn（分散型地震保険）の文脈では以下のような活用可能性があります：

- **IVC**：複数期間にわたる保険料積立・支払い記録の検証
- **証明圧縮**：オンチェーンの検証コスト削減（Ethereum上のガス代節約）
- **zk-Rollup**：多数のポリシーを一括してL2で管理し、L1へ集約
- **Supernova**：異なる種類の計算（Oracle検証、DAO投票記録等）を混在させたIVC

---

*この資料は ZKP MOOC Lecture 10「Recursive SNARKs」（Dan Boneh ほか）の内容を日本語で詳細に解説したものです。*
