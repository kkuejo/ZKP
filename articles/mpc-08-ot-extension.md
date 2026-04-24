---
title: "[MPC入門シリーズ 8/15] OT Extension — IKNP で公開鍵演算をほぼ消す魔法"
emoji: "⚡"
type: "tech"
topics:
  - mpc
  - cryptography
  - oblivioustransfer
  - iknp
  - extension
published: false
---

**日付**: 2026年4月24日
**学習内容**: Article 7 で、Oblivious Transfer (OT) は MPC Complete であること、そして 1 回の OT に公開鍵演算(数 ms)がかかることを見た。しかし MPC は数百万〜数億の OT を要求するため、**素朴な実装では時間が溶ける**。本記事ではこの問題を解く **OT Extension** を扱う。具体的には (1) Beaver (1996) の理論的観察、(2) **IKNP (Ishai-Kilian-Nissim-Petrank 2003)** の matrix-based 構成、(3) Correlation Robust Hash の必要性、(4) Kolesnikov-Kumaresan (KK13) の 1-out-of-$n$ 拡張、(5) Malicious-secure OT Extension、(6) 実装ライブラリと性能を扱う。**$\kappa$ 回の Base OT だけで数億の OT を生成**する魔法を、具体的な行列操作として体得する。

## 0. 本記事の位置づけ

前回見た Naor-Pinkas OT は 1 回あたり数 ms かかる。しかし Yao's GC で $n$ 入力ビットの関数を評価するには $n$ 回の OT が必要。$n = 10^6$ なら $10^3$ 秒 = 17 分かかる。

実用 MPC は 1 秒で終わらせたい。**1 回の OT に平均マイクロ秒以下**が要求される。これを実現するのが **OT Extension**。

**Beaver (1996)** が理論的な発想を示し、**IKNP (2003)** が実用レベルに到達させた。現代のすべての実用 MPC ライブラリ(libOTe、MP-SPDZ、EMP、ABY)は IKNP かその拡張を使う。

本記事の構成:

- **第1章**: なぜ OT Extension が可能か(Beaver の観察)
- **第2章**: IKNP プロトコルの構成
- **第3章**: なぜこれで OT になるか
- **第4章**: Correlation Robust Hash の役割
- **第5章**: Kolesnikov-Kumaresan 拡張と 1-out-of-$n$ OT
- **第6章**: Malicious-secure OT Extension
- **第7章**: 実装と性能
- **第8章**: Q&A

## 1. なぜ OT Extension が可能か

### 1.1 Impagliazzo-Rudich の壁

Impagliazzo-Rudich (1989) が示したのは、**公開鍵暗号を使わずに(OWF だけから)OT を作ることはできない**($P \neq NP$ の一般相対化下で)。つまり**1つ目の OT**には公開鍵演算が必須。

しかし!「**$\kappa$ 個の OT から大量の OT を作る**」ことは、OWF(対称鍵暗号)だけで可能だ、と Beaver (1996) が示した。

### 1.2 Beaver の構成(非実用だが理論的)

Beaver の原論文での構成は:

1. $\kappa$ 個の Base OT を実行
2. それらを使って PRF (擬似ランダム関数) を評価する **GC を MPC で実行**
3. PRF の出力から大量の OT を派生

**理論的には成立**するが、PRF をゲート数の多い GC で評価するため実用的でない。

### 1.3 IKNP の突破口

Ishai-Kilian-Nissim-Petrank (2003) は、**PRF 評価を MPC でやる必要はなく、local に評価して結果を使うだけでいい**という構成を発見。これが今の OT Extension の原型。

## 2. IKNP プロトコルの構成

### 2.1 設定

目標: Sender $\mathcal{S}$ と Receiver $\mathcal{R}$ が、**$m$ 回の 1-out-of-2 OT** を少ないコストで実行したい($m$ は大きい、例: $10^6$)。

$\mathcal{R}$ は選択ビット列 $\mathbf{r} = (r_1, \ldots, r_m) \in \{0,1\}^m$ を持つ。

### 2.2 ステップ 1 — 行列生成(Receiver)

$\mathcal{R}$ は $m \times \kappa$ の 2 つの行列 $T, U$ を作る($\kappa$ は計算安全パラメータ、例: 128):

$$
T, U \in \{0, 1\}^{m \times \kappa}
$$

それらの **$j$ 行目** $\mathbf{t}_j, \mathbf{u}_j$(長さ $\kappa$)は、次を満たす:

$$
\mathbf{t}_j \oplus \mathbf{u}_j = r_j \cdot \mathbf{1}^\kappa = \begin{cases} \mathbf{0}^\kappa & \text{if } r_j = 0 \\ \mathbf{1}^\kappa & \text{if } r_j = 1 \end{cases}
$$

そして $T$ の各行は独立にランダムに選ぶ(したがって $U$ は $T$ と $\mathbf{r}$ から決まる)。

**イメージ**: 各行 $j$ について、「選択ビット $r_j$ を $\kappa$ 個コピーした列 $\oplus$ ランダム列」が $\mathbf{u}_j$。

### 2.3 ステップ 2 — 逆向き Base OT

$\mathcal{S}$ がランダムな **$\kappa$-bit 文字列** $\mathbf{s} \in \{0, 1\}^\kappa$ を選ぶ。

**役割を反転**: ここでは $\mathcal{R}$ が OT の Sender、$\mathcal{S}$ が OT の Receiver として、$\kappa$ 回の Base OT を実行する。

- 第 $i$ 回の Base OT:
  - $\mathcal{R}$ の入力: $T$ の $i$ 列目 $\mathbf{t}^i \in \{0,1\}^m$ と $U$ の $i$ 列目 $\mathbf{u}^i \in \{0,1\}^m$
  - $\mathcal{S}$ の選択ビット: $s_i$
  - $\mathcal{S}$ が受け取るメッセージ: $\mathbf{q}^i \in \{\mathbf{t}^i, \mathbf{u}^i\}$
    - $s_i = 0$ なら $\mathbf{q}^i = \mathbf{t}^i$
    - $s_i = 1$ なら $\mathbf{q}^i = \mathbf{u}^i$

$\mathcal{S}$ は $\kappa$ 個のベクトル $\mathbf{q}^1, \ldots, \mathbf{q}^\kappa$ を得る。これを列として持つ **$m \times \kappa$ 行列 $Q$** を構成。

### 2.4 ステップ 3 — キー観察

$Q$ の $j$ 行目 $\mathbf{q}_j$ は:

$$
\mathbf{q}_j = \mathbf{t}_j \oplus (r_j \cdot \mathbf{s})
$$

つまり:

- $r_j = 0$: $\mathbf{q}_j = \mathbf{t}_j$
- $r_j = 1$: $\mathbf{q}_j = \mathbf{t}_j \oplus \mathbf{s}$

$\mathbf{s}$ は $\mathcal{S}$ だけが知っている。$\mathbf{t}_j$ は $\mathcal{R}$ だけが知っている。

### 2.5 ステップ 4 — OT 生成

$\mathcal{S}$ は Correlation Robust Hash $H$ を使って **2 つの擬似乱数鍵**を計算:

$$
k_j^0 = H(j \| \mathbf{q}_j), \quad k_j^1 = H(j \| \mathbf{q}_j \oplus \mathbf{s})
$$

$\mathcal{R}$ も同様に 1 つだけ計算:

$$
k_j = H(j \| \mathbf{t}_j)
$$

キー観察から、$r_j = b$ なら $k_j = k_j^b$ が成立する:

- $r_j = 0$: $\mathbf{t}_j = \mathbf{q}_j$、よって $k_j = H(j \| \mathbf{q}_j) = k_j^0$
- $r_j = 1$: $\mathbf{t}_j = \mathbf{q}_j \oplus \mathbf{s}$、よって $k_j = H(j \| \mathbf{q}_j \oplus \mathbf{s}) = k_j^1$

### 2.6 ステップ 5 — メッセージ転送

$\mathcal{S}$ がメッセージ $m_j^0, m_j^1$ を送信したい場合:

$$
c_j^0 = k_j^0 \oplus m_j^0, \quad c_j^1 = k_j^1 \oplus m_j^1
$$

を $\mathcal{R}$ に送る。

$\mathcal{R}$ は自分の $k_j = k_j^{r_j}$ を使って:

$$
m_j^{r_j} = c_j^{r_j} \oplus k_j
$$

反対側 $m_j^{1-r_j}$ については、$k_j^{1-r_j}$ を計算できないので分からない。

## 3. なぜこれが OT として成立するか

### 3.1 正当性

ステップ 4 で見たように、$\mathcal{R}$ は $k_j^{r_j}$ を自分の $\mathbf{t}_j$ から正しく計算でき、$m_j^{r_j}$ を得られる。

### 3.2 Receiver の秘密性

$\mathcal{S}$ が $\mathbf{r}$ を学ばないことを示す。

- Base OT では $\mathcal{S}$ が受け取るのは $\mathbf{q}^i = \mathbf{t}^i$ or $\mathbf{u}^i$。どちらが $\mathbf{t}^i$ かは Base OT の秘匿性で不明
- したがって $\mathcal{S}$ から見ると $T$ も $U$ も区別できない
- $\mathbf{t}_j \oplus \mathbf{u}_j = r_j \cdot \mathbf{1}^\kappa$ だから、$T$ が分からなければ $r_j$ も分からない

### 3.3 Sender の秘密性

$\mathcal{R}$ が $m_j^{1-r_j}$ を学ばないことを示す。

- $m_j^{1-r_j}$ は $k_j^{1-r_j} = H(j \| \mathbf{q}_j \oplus \mathbf{s})$(if $r_j = 0$) or $H(j \| \mathbf{q}_j)$(if $r_j = 1$)で隠される
- $\mathcal{R}$ は $\mathbf{q}_j$ を知らず、$\mathbf{s}$ も知らない
- $H$ が **correlation-robust** なら、$\mathcal{R}$ が $k_j^{1-r_j}$ を予測できる確率は無視できる

## 4. Correlation Robust Hash

### 4.1 なぜ普通のハッシュではダメか

$H$ に要求される性質は、**固定のシフト $\mathbf{s}$ に対して $H(x)$ と $H(x \oplus \mathbf{s})$ が独立に見える** こと。

普通の衝突耐性だけでは不十分。IKNP の安全性を証明するには、**$\mathbf{s}$ を含む構造的な「相関」**に対してもハッシュ出力がランダムに見える必要がある。

### 4.2 定義

**Correlation Robust Hash** $H$: 任意の $x_1, \ldots, x_q$ と固定の $\mathbf{s}$ に対して、

$$
\{H(x_i \oplus \mathbf{s})\}_i \approx_c \{U\}_i
$$

ただし $x_i$ は重複しないとする。ここで $U$ は独立な一様ランダム。

### 4.3 実装

実用では **Random Oracle** で代用。AES や SHA-256 を使う:

$$
H(x) = \mathsf{AES}_k(x) \oplus x \quad (\text{Davies-Meyer 構成})
$$

あるいはシンプルに $H(x) = \mathsf{SHA256}(x)$ で OK。

## 5. Kolesnikov-Kumaresan 拡張 — 1-out-of-$n$ OT

### 5.1 KK13 の発想

IKNP では **選択ビット** が 1 bit だった。KK13 (Kolesnikov-Kumaresan 2013) はこれを**符号理論的に一般化**した。

キーアイデア: $r_j$ を「1 bit」ではなく「$\ell$ bit の選択文字列」に拡張。行列の各行 $\mathbf{t}_j \oplus \mathbf{u}_j$ を **誤り訂正符号 $C$ の codeword** $C(r_j)$ にする。

$$
\mathbf{t}_j \oplus \mathbf{u}_j = C(r_j)
$$

- $C$ は minimum distance $\geq \kappa$ の線形符号
- $r_j \in \{0, 1\}^\ell$、$C(r_j) \in \{0, 1\}^\kappa$

これで 1-out-of-$2^\ell$ OT が直接実装できる。IKNP は $\ell = 1$、$C$ が repetition code の特殊ケース。

### 5.2 利点

**Yao's GC で対応値が $2^\ell$ 個ある場合(AES S-box のルックアップなど)**、KK13 で直接 1-out-of-$2^\ell$ OT を使えば、1-out-of-2 OT を $\ell$ 個並べるより効率的。

KK13 は現代の MPC システムで広く採用されている。

### 5.3 Pseudorandom Code と OPRF

Kolesnikov-Kumaresan-Rosulek-Trieu (KKRT16) はさらに **Pseudorandom Code** を提案。これを使うと **Oblivious PRF (OPRF)** が高速化され、**Private Set Intersection (PSI)** が劇的に高速化される。

PSI の Google の商用実装 (2019〜) はこれに近い技術を使っている。

## 6. Malicious-secure OT Extension

### 6.1 課題

Semi-Honest 版の IKNP は、Malicious 敵に対して脆弱:

- Malicious Receiver: 行 $j$ で矛盾する $\mathbf{t}_j, \mathbf{u}_j$ を選ぶことで、$\mathbf{s}$ の情報を漏らす
- Malicious Sender: Base OT で異なる $\mathbf{s}_j$ を行ごとに使う

### 6.2 KOS15 (Keller-Orsini-Scholl 2015)

現代の標準。IKNP にチェック phase を追加し、わずかなオーバーヘッド(5〜10% 増)で Malicious 安全を達成。

### 6.3 ALSZ15 (Asharov-Lindell-Schneider-Zohner 2015)

別アプローチ。IKNP の改良で、特に 1-out-of-$n$ OT の Malicious 化を高速化。

### 6.4 SoftSpokenOT (Roy 2022)

最新研究。通信帯域と計算の両方で改善。

## 7. 実装と性能

### 7.1 数値

2024 年の典型的な実装:

- **Base OT (Naor-Pinkas / Simplest)**: $\kappa = 128$ 回、総計 $\sim 100$ ms
- **OT Extension matrix**: $m = 10^6$ 回、総計 $\sim 200$ ms(LAN)
- **1 OT あたり**: $\sim 200$ ns(ほとんど対称鍵演算のみ)

**超絶速い**。1 秒で $5 \times 10^6$ OT。Naor-Pinkas だけの場合の $10^4$ 倍。

### 7.2 ライブラリ

- **libOTe**: Rindal 作、最速。C++。全バリアント内蔵
- **MP-SPDZ**: OT Extension を内部で使う MPC フレームワーク
- **EMP-toolkit**: C++ MPC、OT は libOTe ベース
- **swanky** (Galois): Rust 実装、ALSZ/KOS 採用

### 7.3 メモリとバッチ

OT Extension の matrix は $m \times \kappa$ bit(例: $10^6 \times 128 / 8 = 16$ MB)。バッチで実行してキャッシュ効率を最大化。

## 8. Q&A

### Q1: なぜ $\kappa = 128$ を使うの? 小さくできない?

$\kappa$ は計算安全パラメータ。攻撃者の総当たり計算量を $2^\kappa$ 以下に抑える。128 bit は現代の標準(量子耐性を気にするなら 256 bit)。

### Q2: OT Extension の通信量は?

$m$ 回の OT で総通信量 $O(m \kappa)$ bit。1 OT あたり $O(\kappa) = 128$ bit。オリジナル OT($\sim$ KB/回)より 60 倍以上削減。

### Q3: IKNP は Random Oracle 必須?

**厳密には correlation robust hash が必要**。RO で代用するのが実装的に簡単。標準仮定(AES-PRF など)で示した変種もあるが、実装は RO 仮定が主流。

### Q4: Post-Quantum OT Extension は?

**研究中**。LWE ベースの OT Extension が近年登場。性能は古典比で 10 倍程度遅いが、改善進行中。

### Q5: 1-out-of-2 と 1-out-of-$n$ のどちらを使う?

用途次第:
- Yao's GC の入力配送: 1-out-of-2 で十分
- AES S-box: 1-out-of-$2^8 = 256$ を直接
- PSI: OPRF(1-out-of-$\infty$ の亜種)

### Q6: Base OT は何回必要?

**$\kappa$ 回**。標準的に 128 or 256。これが OT Extension の「種」。

### Q7: OT Extension は後で追加生成できる?

**ある程度まで**。もう 1 回 IKNP を走らせれば新しいバッチを得られる。Base OT は再利用できない(または Pseudorandom Base OT が必要)。

### Q8: Random OT (ROT) との違いは?

**ROT**: メッセージがランダムで、両者が共通の $\{m_0, m_1\}$ を知る(Receiver は $m_b$ のみ)
**Chosen-message OT**: Sender がメッセージを選ぶ

ROT は事前計算用。Chosen-message は「オンライン」で ROT をマスクしたものを送る($c = m \oplus k$)。

### Q9: OT Extension が壊れる攻撃はある?

**Semi-Honest 版を Malicious 敵に使うと壊れる**(行ごとに矛盾する選択)。これが KOS15 の動機。Malicious 版を使う限り安全。

### Q10: 実装で気をつけることは?

- **乱数生成**: 暗号学的に安全な乱数(Rust なら `rand::rngs::OsRng`)
- **メモリ安全**: Rust か C++ の注意深い実装。libOTe / swanky を推奨
- **AES-NI 活用**: CPU ハードウェア命令で 10 倍速い
- **パラメータ**: $\kappa \geq 128$ 厳守

## 9. まとめ

### 本記事で学んだこと

- **OT Extension は $\kappa$ 回の Base OT から大量の OT を作る**。公開鍵演算を劇的に削減
- **IKNP (2003)** が実用レベルの最初の構成。$m \times \kappa$ matrix を使う
- 核心: **Receiver が $\mathbf{t}_j \oplus \mathbf{u}_j = r_j \cdot \mathbf{1}$ を仕込み、Base OT で Sender が $\mathbf{q}_j = \mathbf{t}_j \oplus r_j \cdot \mathbf{s}$ を得る**
- **Correlation Robust Hash** が安全性の鍵。RO で代用
- **KK13 / KKRT16**: 1-out-of-$n$ OT と OPRF/PSI への拡張
- **KOS15**: Malicious 安全版、オーバーヘッド 5〜10%
- **性能**: 1 OT 200 ns 以下。1 秒で数百万〜数千万 OT
- **ライブラリ**: libOTe、MP-SPDZ、EMP、swanky

### 次の記事(Article 9)へ

Article 5〜8 で **Shamir 秘密分散、BGW、OT、OT Extension** を揃えた。ここから **2PC の花形 Yao's Garbled Circuits** に入る。

- Boolean 回路 → garbled circuit への変換
- wire labels と garbled tables
- Generator / Evaluator の役割
- OT で入力配送
- Semi-Honest 安全性の直感

### 3行サマリ

- **OT Extension = $\kappa$ 回の Base OT + 対称鍵演算で数百万 OT を生成する魔法**
- **IKNP のキー技: Receiver の matrix $T, U$ で $\mathbf{t}_j \oplus \mathbf{u}_j = r_j \cdot \mathbf{1}$ を仕込む**
- **実用 MPC の心臓部 — Yao's GC、GMW、SPDZ のすべてが内部で OT Extension を呼ぶ**

---

## 参考文献

- Yuval Ishai, Joe Kilian, Kobbi Nissim, Erez Petrank. *Extending Oblivious Transfers Efficiently*. CRYPTO 2003.
- Donald Beaver. *Correlated Pseudorandomness and the Complexity of Private Computations*. STOC 1996.
- Vladimir Kolesnikov, Ranjit Kumaresan. *Improved OT Extension for Transferring Short Secrets*. CRYPTO 2013.
- Marcel Keller, Emmanuela Orsini, Peter Scholl. *Actively Secure OT Extension with Optimal Overhead*. CRYPTO 2015.
- Gilad Asharov, Yehuda Lindell, Thomas Schneider, Michael Zohner. *More Efficient Oblivious Transfer Extensions with Security for Malicious Adversaries*. EUROCRYPT 2015.
- Vladimir Kolesnikov, Ranjit Kumaresan, Mike Rosulek, Ni Trieu. *Efficient Batched Oblivious PRF with Applications to Private Set Intersection*. ACM CCS 2016.
- Lance Roy. *SoftSpokenOT: Quieter OT Extension from Small-field Silent VOLE in the Minicrypt Model*. CRYPTO 2022.
