---
title: "[MPC入門シリーズ 13/15] Malicious セキュリティ — Cut-and-Choose、SPDZ、Authenticated Garbling"
emoji: "🛡️"
type: "tech"
topics:
  - mpc
  - cryptography
  - malicious
  - cutandchoose
  - spdz
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事では、**Malicious 敵**に対して安全な MPC プロトコルを構築する 3 大アプローチを扱う: (1) **Cut-and-Choose** — Yao's GC を $s$ 個作って一部を検査、(2) **Authenticated Secret Sharing** — SPDZ / BDOZ の MAC 付きシェア、(3) **Authenticated Garbling** — Wang-Ranellucci-Katz 2017 の新しい決定版。それぞれの設計思想、コスト、使い分けを比較し、現代の MPC が Malicious 攻撃者に対してどう防御しているかを俯瞰する。また **Input consistency**、**Selective Failure attack**、**LEGO** などの技術的トピックも扱い、最後に **実運用での選び方** を議論する。

## 0. 本記事の位置づけ

Article 9〜12 で見たプロトコルは基本的に **Semi-Honest** 安全だった。しかし実運用では **Malicious** 敵(任意の逸脱、偽メッセージ、結託)を想定する必要がある(Article 3 の脅威モデル)。

Semi-Honest を Malicious に昇格させるには、大きく 3 つのアプローチがある:

1. **Cut-and-Choose**: GC を検査することで不正を検出(主に 2PC)
2. **Authenticated Secret Sharing**: シェアに MAC を付けて改ざんを検出(SPDZ 系、多者向け)
3. **Authenticated Garbling**: GC と MAC を融合した最新手法(2017 年以降)

本記事の構成:

- **第1章**: Malicious の脅威
- **第2章**: Cut-and-Choose(GC の Malicious 化)
- **第3章**: Input Consistency と Selective Failure
- **第4章**: LEGO と batch cut-and-choose
- **第5章**: SPDZ / BDOZ / MASCOT(SS の Malicious 化)
- **第6章**: Authenticated Garbling(2017〜)
- **第7章**: 性能比較と選び方
- **第8章**: Q&A

## 1. Malicious の脅威 — 何が起こりうるか

### 1.1 GC の脆弱性

Yao's GC で Malicious Generator が仕掛けられる攻撃:

**偽の garbled table**:
- 例: AND ゲートを実は OR として garble
- 出力 label のデコード表で虚偽を仕掛ける
- Evaluator は気づかずに実行

**Selective Failure 攻撃**:
- 入力 label を1つだけ「壊す」
- Evaluator がその入力を選ぶと復号失敗 → プロトコル中止
- Generator は**中止が起きた = Evaluator の入力 = その値**と推論可能

**出力偽装**:
- デコード表で任意のビットに書き換えさせる

### 1.2 SS ベース MPC の脆弱性

**偽のシェア**:
- Malicious プレイヤーが嘘のシェアを送る
- 復元値が間違う
- 他のプレイヤーには検出できない

**MAC なしの通信**:
- 通信路で改ざんされても気づかない

これらをすべて防ぐのが **Malicious-secure MPC** の目標。

## 2. Cut-and-Choose — GC の Malicious 化

### 2.1 基本アイデア

**複数の GC を生成し、一部を検査する**:

1. Generator が $s$ 個の GC を作る(例: $s = 40$)
2. Evaluator がランダムに半分を選ぶ
3. Generator は選ばれた GC の**すべての乱数シード**を公開
4. Evaluator はそれらが正しく garbled されているか検証
5. 残りの GC を実際に評価し、**多数決**で出力を決定

### 2.2 なぜ動くか

Malicious Generator が偽 GC を混ぜた場合:

- 検査されると**即バレ**
- 検査されない方に忍ばせても、**多数決で潰される**(正しい GC が過半数必要)

攻撃者が成功する確率は $\binom{s}{s/2}^{-1}$ に近いが、より精密な解析で **replication factor $s \approx 3.12 \lambda$** で $2^{-\lambda}$ 安全。

### 2.3 コスト

1 関数評価で $s$ 個の GC を作る **→ 通信・計算が $s$ 倍**。$s = 40$ で 40 倍のオーバーヘッド。

### 2.4 改良

- **shelat-Shen (2011)**: 2 universal hash で input consistency 改善
- **Lindell (2013)**: **Input recovery** で replication factor を $\lambda$ に圧縮
- **Brandão (2013)**: 同時期独立の改良
- **LEGO (Frederiksen et al. 2013)**: 個別ゲート単位の cut-and-choose

## 3. Input Consistency と Selective Failure

### 3.1 Input Consistency

**問題**: Generator が異なる GC で異なる入力 labels を Evaluator に渡すと、「本当の出力」が分からなくなる。

**解決**: すべての GC で Evaluator が受け取る入力 label が**同じ意味**であることを保証。

**shelat-Shen (2011) の方法**: 2-universal hash $H$ を追加し、計算する関数を $(f(x, y), H(x, r))$ に拡張。Generator が GC 間で $x$ を変えると $H(x, r)$ が異なる。ランダム $H$ で検出確率が上がる。

FreeXOR と併用する場合、$H$ を XOR 回路で構成すれば**ほぼ無料**で追加できる。

### 3.2 Selective Failure Attack

**攻撃**: Generator が OT で渡す入力 labels の片方を「壊す」:

- $k_w^0$ は正常、$k_w^1$ は無効な値
- Evaluator が $w = 1$ を選ぶと復号失敗 → abort
- abort 発生 = Evaluator の入力 bit = 1 と判明

**解決 (Lindell-Pinkas 2007)**: **$k$-probe-resistant matrix** で Evaluator の入力をランダマイズ。

- Evaluator は真入力 $y$ を **ランダムに拡張** $\tilde{y}$ として、$y = M \tilde{y}$($M$ は公開行列)
- $k$ 個以下の selective failure では漏れるビットがない
- $k$ より多いと abort 確率がほぼ 1 = 検出

FreeXOR と併用すれば、$y \mapsto M\tilde{y}$ は XOR 演算で無料。

## 4. LEGO と Batch Cut-and-Choose

### 4.1 Batch Cut-and-Choose

**Lindell-Riva (2014)**: 同じ関数を複数回実行する場合、**global な cut-and-choose**で amortized cost を削減。

例: $N$ 回の評価で、$N \times \lambda$ ではなく $N \times (2 + \lambda/\log N)$ 程度の replication。

### 4.2 LEGO (Frederiksen-Jakobsen-Nielsen-Nordholt-Orlandi 2013)

**発想**: GC 全体ではなく、**個別ゲート単位**で cut-and-choose。

- Generator が膨大な数の NAND garbled gates を生成
- 一部を検査、残りを**バケット**にまとめる
- 各バケットが **1 つの論理ゲート**を構成(多数決で fault tolerance)
- バケットをつなぐために **soldering**(接合)

LEGO は**大規模回路で効率的**だが、実装が複雑。

### 4.3 MiniLEGO, TinyLEGO

連続する改良で性能向上:

- **MiniLEGO (2013)**: Standard assumptions
- **TinyLEGO (2015)**: 1 gate per ciphertext
- **DUPLO (Kolesnikov et al. 2017)**: Component-level cut-and-choose

## 5. SPDZ / BDOZ / MASCOT — SS の Malicious 化

### 5.1 Authenticated Shares

Article 12 で紹介したように、SPDZ はシェアに MAC を付ける:

$$
[v]_i = (v^{(i)}, m^{(i)}) \quad \text{where } m^{(i)} \text{ is a MAC on } v \text{ with key } \alpha
$$

**MAC 確認**: 値を reveal する前に全員で MAC をチェック。

### 5.2 Global MAC Key

SPDZ では global MAC key $\alpha$ を秘密分散:

$$
\alpha = \sum_i \alpha^{(i)}
$$

各プレイヤー $P_i$ は $\alpha^{(i)}$ を保持。

**MAC 検証の特性**:
- 多項式時間の攻撃者が MAC を偽造する確率は $1/|\mathbb{F}|$(情報理論的)
- $|\mathbb{F}| = 2^{128}$ なら統計的に無視できる

### 5.3 Malicious Triples

Beaver triple も MAC 付き:

$$
[a], [b], [c] \to ([a], [\alpha a], [b], [\alpha b], [c], [\alpha c])
$$

これの生成が **SPDZ のオフラインフェーズ**。重い計算(SHE または MASCOT の OT)。

### 5.4 MASCOT (Keller-Orsini-Scholl 2016)

Triple 生成を OT Extension で高速化。SHE 不要。現代の SPDZ 実装のデファクト。

### 5.5 Overdrive / Overdrive2k

LWE-based SHE での triple 生成の高速化研究。整数環 $\mathbb{Z}_{2^k}$ 上の MPC (SPDZ-2k) でよく使われる。

## 6. Authenticated Garbling — 2017 年の決定版

### 6.1 背景

Cut-and-Choose は replication で重い。SPDZ は多者対応だが 2PC では GC が有利。**GC の Malicious 化で replication なしの方法はないか?**

**Wang-Ranellucci-Katz (ACM CCS 2017)**: SPDZ 的な MAC 付きシェアと GC を融合した **Authenticated Garbling** を提案。

### 6.2 構造

- 回路の各 wire に、**BDOZ 的な authenticated share** を用意
- GC の構成に MAC を織り込む
- 偽の garbled gate は MAC 検証で検出

### 6.3 性能

- **2PC Malicious**: Cut-and-Choose の 5 倍高速
- **多者 Malicious**: SPDZ ベースを上回る場合も

EMP-toolkit に実装あり、研究最前線で広く使われる。

### 6.4 Authenticated BMR

多者版 (Wang-Ranellucci-Katz 2017b) もある。**定数ラウンド、多者、Malicious** を同時に達成。BMR + SPDZ 的 MAC の融合。

## 7. 性能比較と選び方

### 7.1 比較表(2024 年時点)

| プロトコル | 用途 | Malicious | 速度(AES 1 回 LAN) | 備考 |
|---|---|---|---|---|
| Yao's GC (Semi-Honest) | 2PC | No | $\sim$ 2 ms | 基準 |
| Cut-and-Choose (s=40) | 2PC | Yes | $\sim$ 80 ms | replication 重い |
| LEGO | 2PC large | Yes | 10-50 ms | 大規模有利 |
| SPDZ + MASCOT | 多者 | Yes | 数十 ms(offline 除く) | オフライン重い |
| Authenticated Garbling (WRK17) | 2PC/多者 | Yes | 10-15 ms | 最新決定版 |

### 7.2 選び方

- **2PC, Semi-Honest, LAN**: Yao's GC(Half-Gates)
- **2PC, Malicious, LAN**: Authenticated Garbling
- **2PC, Malicious, WAN**: Authenticated Garbling(通信効率良)
- **多者 ($n \geq 3$), Malicious**: MASCOT + SPDZ or Authenticated BMR
- **Honest Majority 3PC**: Sharemind / Araki et al.(超高速)

### 7.3 実装

- **EMP-toolkit**: Authenticated Garbling、Wang et al. 著者陣がメンテ
- **MP-SPDZ**: SPDZ / MASCOT / BDOZ、Python-like
- **SCALE-MAMBA**: 商用志向、SPDZ 系

## 8. Q&A

### Q1: Malicious 対応で必ず遅くなるの?

**ほぼ必ず**。最良で $2\times$、素朴では $40\times$。ただし Authenticated Garbling で **$5\sim 10 \times$** まで改善された。

### Q2: Covert / PVC は実用的?

**特定シナリオで**。銀行間や政府間など、評判リスクが効く環境。Article 3 の PVC は公開証拠付きで、監査可能性が必要な場合に有力。

### Q3: OT も Malicious 対応が必要?

**必要**。Cut-and-Choose や Authenticated Garbling の内部で Malicious-secure OT(Keller-Orsini-Scholl 2015)を使う。

### Q4: GMW compiler は実用?

**ほぼ使われない**。理論的には任意の Semi-Honest を Malicious に昇格できるが、**コストが重い**。特定プロトコルの Malicious 版を直接設計する方が効率的。

### Q5: UC-secure は追加コスト?

**ある程度**。UC の straight-line simulator 要求で、通常の rewinding ベースより難しい。ただし最新プロトコル(WRK17 等)は UC-secure を達成。

### Q6: Malicious 安全の MPC はどこまで実装可能?

**任意の関数**(算術・Boolean)。ただし実装の複雑さは高い。多くのライブラリが「**Malicious 保証**」をオプションで提供。

### Q7: 量子耐性 + Malicious は?

**研究途上**。OT を Lattice-based に置き換え、MAC も量子耐性ハッシュを使う設計が提案されているが、性能は古典比で大幅に劣る。

### Q8: Adaptive Malicious は?

**さらに難しい**。Equivocable commitments が必要。理論的には可能だが、実装は限定的。本シリーズは Static corruption に集中。

### Q9: どのくらいの規模まで実用?

2024 年時点:
- **2PC Malicious**: $10^7$ gates まで実用(数秒〜数十秒)
- **多者 Malicious**: $10^6$ gates まで実用
- **Authenticated Garbling**: $10^8$ gates も射程内

### Q10: 実装で気をつけること?

- **乱数**: 暗号学的に安全な RNG 必須
- **MAC サイズ**: $|\mathbb{F}| \geq 2^{128}$
- **Offline/Online 分離**: 本番で triple 枯渇しないよう管理
- **Abort 処理**: MAC 不整合時の適切な中止
- **タイムアウト**: Malicious 敵がハングさせる可能性

## 9. まとめ

### 本記事で学んだこと

- **Malicious の脅威**: 偽 GC、selective failure、偽シェア、MAC なし通信
- **Cut-and-Choose**: $s$ 個の GC で検査と多数決。replication factor $\sim 40$
- **Input Consistency**: 2-universal hash で複数 GC 間の入力一致を保証
- **Selective Failure**: $k$-probe-resistant matrix でランダマイズ
- **LEGO**: 個別ゲート単位の cut-and-choose、大規模で有利
- **SPDZ / BDOZ / MASCOT**: 多者 Authenticated SS、Beaver triples + MAC
- **Authenticated Garbling (WRK17)**: 2017 年の決定版、GC と MAC の融合

### 次の記事(Article 14)へ

理論を十分に学んだので、次は **実装とツール** の回:

- **MP-SPDZ**: Python-like、多プロトコル
- **ABY / ABY3**: GC + GMW + Arithmetic の混在
- **EMP-toolkit**: C++、最速クラス
- **Obliv-C, Frigate**: DSL と compiler
- **Sharemind**: 商用 3PC
- 実際にコードを書いて MPC を動かす方法

### 3行サマリ

- **Malicious 化の 3 アプローチ: Cut-and-Choose / Authenticated SS / Authenticated Garbling**
- **Cut-and-Choose は replication 重い($\sim 40\times$)、SPDZ は MAC で軽量、WRK17 が最新決定版**
- **選び方は 2PC vs 多者、LAN vs WAN、Honest vs Dishonest majority で決まる**

---

## 参考文献

- Yehuda Lindell, Benny Pinkas. *An Efficient Protocol for Secure Two-Party Computation in the Presence of Malicious Adversaries*. EUROCRYPT 2007.
- abhi shelat, Chih-Hao Shen. *Two-Output Secure Computation with Malicious Adversaries*. EUROCRYPT 2011.
- Yehuda Lindell. *Fast Cut-and-Choose Based Protocols for Malicious and Covert Adversaries*. CRYPTO 2013.
- Tore Frederiksen, Thomas Jakobsen, Jesper Buus Nielsen, Peter Sebastian Nordholt, Claudio Orlandi. *MiniLEGO: Efficient Secure Two-Party Computation from General Assumptions*. EUROCRYPT 2013.
- Yehuda Lindell, Ben Riva. *Cut-and-Choose Yao-Based Secure Computation in the Online/Offline and Batch Settings*. CRYPTO 2014.
- Xiao Wang, Samuel Ranellucci, Jonathan Katz. *Authenticated Garbling and Efficient Maliciously Secure Two-Party Computation*. ACM CCS 2017.
- Xiao Wang, Samuel Ranellucci, Jonathan Katz. *Global-Scale Secure Multiparty Computation*. ACM CCS 2017.
- Marcel Keller, Emmanuela Orsini, Peter Scholl. *MASCOT: Faster Malicious Arithmetic Secure Computation with Oblivious Transfer*. ACM CCS 2016.
