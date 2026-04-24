---
title: "[MPC入門シリーズ 7/15] Oblivious Transfer (OT) — MPC の万能プリミティブ"
emoji: "📬"
type: "tech"
topics:
  - mpc
  - cryptography
  - oblivioustransfer
  - naorpinkas
  - publickey
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事では、MPC の**最も重要なビルディングブロック**の1つ、**Oblivious Transfer (OT)** を扱う。OT は Sender が2つの秘密 $m_0, m_1$ を持ち、Receiver が選択ビット $b$ で $m_b$ だけを受け取る機能で、**Sender は Receiver の選択 $b$ を知らず、Receiver は $m_{1-b}$ を知らない**という二重の秘匿性を持つ。Kilian (1988) は「OT があれば任意の 2PC/MPC が作れる」と示し、以来 OT は MPC の**万能建材**と呼ばれている。本記事では (1) OT の定義と変種、(2) Kilian の Completeness 定理、(3) Naor-Pinkas による公開鍵ベース OT の構成、(4) DDH 仮定に基づく証明の直感、(5) 応用(Yao's GC、GMW、PSI)を順に見る。

## 0. 本記事の位置づけ

前回までで **Secret Sharing ベースの MPC**(Shamir + BGW)を見た。これは Honest Majority では非常に効率的だが、**2者間 (2PC) や Dishonest Majority では使えない**(シェアを半分以下の人数で復元できないため)。

Dishonest Majority MPC の核心は **Oblivious Transfer (OT)** という別の道具。**Yao's Garbled Circuits、GMW プロトコル、SPDZ**などすべての dishonest majority MPC が、内部で OT を無数に呼び出す。

本記事の構成:

- **第1章**: OT の定義と変種
- **第2章**: Kilian の Completeness 定理(OT からMPCを作る)
- **第3章**: 公開鍵ベース OT (Naor-Pinkas)
- **第4章**: DDH 仮定と安全性の直感
- **第5章**: Bellare-Micali などの別方式
- **第6章**: 応用と効率の課題
- **第7章**: Q&A

## 1. OT の定義

### 1.1 1-out-of-2 OT

**登場人物**: Sender $\mathcal{S}$ と Receiver $\mathcal{R}$。

**入力**:
- $\mathcal{S}$: 2つのメッセージ $m_0, m_1 \in \{0,1\}^\ell$
- $\mathcal{R}$: 選択ビット $b \in \{0, 1\}$

**出力**:
- $\mathcal{S}$: 何もなし
- $\mathcal{R}$: $m_b$

**セキュリティ要件**:

- **Receiver の秘密性**: $\mathcal{S}$ は $b$ について何も学ばない
- **Sender の秘密性**: $\mathcal{R}$ は $m_{1-b}$ について何も学ばない

```mermaid
sequenceDiagram
    participant S as Sender
    participant R as Receiver
    Note over S,R: S has (m0, m1), R has b
    S<-->R: OT プロトコル
    Note over R: 出力 m_b
    Note over R: m_{1-b} は不明
    Note over S: b は不明
```

### 1.2 変種

**1-out-of-n OT**: Sender が $n$ 個のメッセージ $m_0, \ldots, m_{n-1}$、Receiver が選択 $b \in \{0, \ldots, n-1\}$、$\mathcal{R}$ は $m_b$ のみ取得。

**$k$-out-of-$n$ OT**: Receiver が $k$ 個を選んで受け取る。

**Random OT (ROT)**: メッセージは事前にランダム。両者の合意済み擬似乱数を使う OT で、**前計算**(Article 8)でよく使う。

**Correlated OT**: Sender のメッセージに相関がある($m_0 = r, m_1 = r \oplus \Delta$)。OT Extension の内部で使う。

**String OT**: メッセージが $\ell$-bit 文字列(上で既に述べた)。1-bit OT を $\ell$ 回並列実行するのと等価だが、より効率的な構成あり。

### 1.3 Functionality 記法

形式的に、1-out-of-2 OT のファンクショナリティ $\mathcal{F}_{\mathsf{OT}}$ は:

```
F_OT:
  Receive (m0, m1) from Sender
  Receive b from Receiver
  Output m_b to Receiver
  Output nothing to Sender
```

このファンクショナリティを安全に実現する通信プロトコルが **OT プロトコル**。

## 2. Kilian の Completeness 定理

### 2.1 定理

**Kilian (1988)**: **OT があれば、任意の関数を2者間で安全に計算できる**(Semi-Honest / Malicious 両方)。

つまり OT は **MPC Complete**。逆に、OT 無しで2者間 MPC を作ることは(OWFs のみでは)できない(Impagliazzo-Rudich 1989)。

### 2.2 直感

Yao's Garbled Circuits(Article 9)は OT を前提とする:

- Generator (Sender) が Boolean 回路を garbling する
- Evaluator (Receiver) は自分の入力に対応する**garbled label** を OT で取得
- Evaluator は $m_0$(input=0 のラベル)と $m_1$(input=1 のラベル)のどちらか片方だけ受け取る
- Generator は Evaluator がどちらを選んだか知らない(秘匿性保存)

GMW プロトコル(Article 11)でも、AND ゲートごとに 1-out-of-4 OT を 1 回呼ぶ。

### 2.3 「万能建材」の意味

1 ビットの OT を大量に実行すれば、任意の関数が計算できる。**MPC の究極の「原子」が OT** と言っていい。だから OT 自体を高速化することが、MPC 全体の高速化につながる(Article 8 の OT Extension)。

## 3. 公開鍵ベース OT — Naor-Pinkas

### 3.1 背景

OT を実装するには、暗号学的な非対称性が必要。**公開鍵暗号**が典型的な基盤。

Naor-Pinkas (2001) は **Diffie-Hellman (DH)** を使った効率的な OT を設計。今も多くの OT 実装で採用されている。

### 3.2 前提

巡回群 $G$(位数 $q$、生成元 $g$)で **Decisional Diffie-Hellman (DDH) 仮定**:

> **$(g^a, g^b, g^{ab})$ と $(g^a, g^b, g^c)$($c$ はランダム)は多項式時間で区別不能**

楕円曲線群(Curve25519 など)で成立すると信じられている。

### 3.3 Naor-Pinkas プロトコル

**セットアップ**: $G, q, g$ は公開。さらに「$\log_g C$ が誰にも分からない」値 $C \in G$ を公開(nothing-up-my-sleeve で生成)。

**プロトコル**:

1. **Receiver**: ランダム $x \in \mathbb{Z}_q$ を選ぶ。選択ビット $b \in \{0, 1\}$ に応じて:

$$
(h_0, h_1) = \begin{cases} (g^x, C) & \text{if } b = 0 \\ (C, g^x) & \text{if } b = 1 \end{cases}
$$

$(h_0, h_1)$ を Sender に送る。**キー観察**: $h_0 \cdot h_1 = C \cdot g^x$ は常に成立。

2. **Sender**: $h_0 \cdot h_1$ が $C \cdot g^x$ の形になっていることを確認(ここでは省略されているチェックあり)。ランダム $r_0, r_1 \in \mathbb{Z}_q$ を選び、次を計算:

$$
e_i = \left( g^{r_i}, \; H(h_i^{r_i}) \oplus m_i \right) \quad (i = 0, 1)
$$

ここで $H$ は擬似ランダムハッシュ。$(e_0, e_1)$ を Receiver に送る。

3. **Receiver**: $b$ に対応する $e_b = (u_b, v_b)$ から、自分が知る秘密 $x$ を使って:

$$
m_b = v_b \oplus H(u_b^x)
$$

### 3.4 なぜ動くか

Receiver の選択 $b = 0$ のときを追う:

- $h_0 = g^x$、$h_1 = C$
- Sender が計算: $e_0 = (g^{r_0}, H((g^x)^{r_0}) \oplus m_0) = (g^{r_0}, H(g^{x r_0}) \oplus m_0)$
- Receiver は $u_0 = g^{r_0}$ と $x$ を知っているので $u_0^x = g^{x r_0}$ を計算可能
- $H(u_0^x) = H(g^{x r_0})$ なので $v_0$ との XOR で $m_0$ が復元できる

一方 $e_1 = (g^{r_1}, H(C^{r_1}) \oplus m_1)$ について:

- Receiver は $C^{r_1}$ を計算したいが、$r_1$ を知らず、$\log_g C$ も知らない
- DDH 仮定の下で $C^{r_1}$ は **$m_1$ を隠すのに十分ランダム**

### 3.5 セキュリティの直感

**Receiver の秘密性**: $(h_0, h_1) = (g^x, C)$ or $(C, g^x)$ のどちらか。どちらのケースも「片方が一様ランダム($g^x$)、もう片方が公開定数($C$)」で分布上は同じ。**Sender から見て $b$ は完全に隠れる**。

**Sender の秘密性**: DDH 仮定により、Receiver は $m_{1-b}$ を平文で学べない。形式的には「Receiver が $m_{1-b}$ を復元できる ⇒ DDH を破れる」に帰着する。

### 3.6 コストの目安

- **計算**: 両者で $\sim 2$ 回の冪乗(exponentiation)。$\sim 1$ ms 程度(256 bit 曲線)
- **通信**: $O(1)$ 群要素 + 暗号文 2 つ。$\sim 1$ KB 程度

**問題**: MPC で **数百万回** OT を呼ぶ必要があると、$10^6 \times 1$ ms $= 1000$ 秒 = 17 分。**遅すぎる**。これが OT Extension(Article 8)の動機。

## 4. Semi-Honest 安全性の証明スケッチ

### 4.1 シミュレータの構成

Article 2 で述べた Real/Ideal パラダイムで、OT の安全性を証明する。

**Sender が腐敗したケース**:

- Ideal: $\mathcal{F}_{\mathsf{OT}}$ に $(m_0, m_1)$ を渡すだけ
- Real: Sender は $(h_0, h_1)$ を Receiver から受け取る
- $\mathsf{Sim}_{\mathcal{S}}$: $(m_0, m_1)$ しか知らないが、「ランダムな $(h_0, h_1)$ with $h_0 \cdot h_1 = C \cdot g^x$ for some random $x$」を生成して Sender に送ればよい

Real の $(h_0, h_1)$ と Sim の生成物は分布上同じ($b$ 両方で一様)。**Perfect Simulation**。

**Receiver が腐敗したケース**:

- Ideal: $\mathcal{F}_{\mathsf{OT}}$ から $m_b$ だけ受け取る
- Real: Receiver は $(e_0, e_1)$ を受け取るが、復号できるのは $e_b$ だけ
- $\mathsf{Sim}_{\mathcal{R}}$: $m_b$ は知っているので $e_b$ は正しく作れる。$e_{1-b}$ は **ダミー暗号文(ランダム)** にする

Real と Sim の違いは $e_{1-b}$ だけ。DDH 仮定により区別不能 → **Computational Simulation**。

### 4.2 Malicious への拡張

- Sender が Malicious: $(h_0, h_1)$ が有効かチェック(例: $h_0 \cdot h_1 = C \cdot g^x$ を示す ZKP)
- Receiver が Malicious: $\log_g h_b$ を知っていることを証明(Knowledge of DL)

Peikert-Vaikuntanathan-Waters (2008) の UC-secure OT が定番。Article 13 で簡単に触れる。

## 5. 別の OT 構成 — Bellare-Micali、Simplest OT

### 5.1 Bellare-Micali (1989)

最初の効率的な公開鍵ベース OT。Naor-Pinkas と似た DH ベース。

### 5.2 Simplest OT (Chou-Orlandi 2015)

**「一番単純な OT」**と呼ばれる構成。DDH 仮定で Semi-Honest 安全。

- **プロトコル**:
  1. Sender: 秘密 $y$ を選び $S = g^y$ を送る
  2. Receiver: 秘密 $x$ を選び $R = g^x$(if $b=0$)or $S \cdot g^x$(if $b=1$)を送る
  3. Sender: $k_0 = H(R^y)$, $k_1 = H((R/S)^y)$ で $m_0, m_1$ を暗号化して送る

Receiver は $k_b = H(S^x)$ を自分で計算できる。実装が簡単で、libOTe などで採用。

### 5.3 McQuoid-Rosulek-Roy (2020)

**Endemic OT** という新パラダイム。MPC-friendly で 1 round。Correlated OT 向け。

OT 研究は今も進行中で、用途ごとに微妙に異なる最適化がされている。

## 6. 応用 — OT はどこで使われるか

### 6.1 Yao's Garbled Circuits (Article 9 で詳述)

Yao's GC では、**Evaluator の入力を Generator から取得**するのに OT を使う:

- 各入力 wire について、Generator は 2 つの garbled label $k_0, k_1$ を持つ
- Evaluator は自分の入力 $b$ に対応する $k_b$ だけ受け取る
- Generator は Evaluator の入力 $b$ を知らない

### 6.2 GMW プロトコル (Article 11 で詳述)

GMW では **AND ゲートごとに 1 回の 1-out-of-4 OT** を実行。

- Boolean 回路を XOR secret sharing で評価
- XOR は局所計算
- AND は OT 必須

### 6.3 Private Set Intersection (PSI)

2 者が集合 $A, B$ の交差 $A \cap B$ だけを知りたい。OT ベースの PSI プロトコル (KKRT16, PaXoS など) が最速クラス。

### 6.4 Oblivious RAM (ORAM)

MPC で配列の index-private access をする際、OT が基本要素。

### 6.5 閾値暗号

Threshold ECDSA の部分プロトコルで OT が使われる(Lindell 2017, Doerner et al. 2019)。

## 7. 実装と性能

### 7.1 ライブラリ

- **libOTe** (Peter Rindal): 最速の OT/OT Extension 実装。C++
- **MP-SPDZ**: 各種 OT プロトコル内蔵、Python-like DSL
- **EMP-toolkit**: C++ MPC フレームワーク

### 7.2 性能数値(2024 年時点)

- **Base OT (Naor-Pinkas / Simplest)**: 数百μs〜数 ms/回。公開鍵演算なので重い
- **OT Extension** (Article 8): **数千万回 OT/秒**。Base OT は $\kappa$ 回だけ
- 128 bit 安全: Curve25519 が標準

### 7.3 実装上の注意

- **同時 OT の並列化**: バッチで実行して通信遅延を隠す
- **Malicious-secure OT**: Keller-Orsini-Scholl (2015) が標準
- **乱数生成**: 暗号学的に安全な乱数必須

## 8. Q&A

### Q1: OT と公開鍵暗号はどう違うの?

**公開鍵暗号**: Alice が暗号文を作り Bob が復号。Bob はすべての情報を得る。
**OT**: Sender が 2 つの秘密、Receiver は片方だけ、しかも Sender はどちらが選ばれたか不明。

「双方向の秘匿性」が OT 特有。

### Q2: OT はなぜ MPC Complete なのか?

Kilian (1988) による構成的証明。大雑把には: OT は「Boolean AND を秘匿計算」できる最小プリミティブ。AND + XOR で任意の Boolean 関数が作れる。GMW や Yao's GC がこれを具現化。

### Q3: Naor-Pinkas 以外の重要な構成は?

- **Bellare-Micali (1989)**: 最古の効率的 OT
- **Simplest OT (Chou-Orlandi 2015)**: 単純で高速
- **Peikert-Vaikuntanathan-Waters (2008)**: UC-secure Malicious
- **McEliece ベース**: Post-quantum 候補
- **Lattice ベース (LWE)**: Post-quantum 候補

### Q4: Post-Quantum OT はある?

**ある**。Kyber/CRYSTALS、McEliece ベース、Ring-LWE ベースなど。パフォーマンスは従来比 10〜100 倍遅いが、実用化が進行中。

### Q5: 2 者間で信頼できる第三者を使えば OT は自明では?

**自明**。Sender と Receiver が TTP に送るだけ。OT の面白さは **TTP 無しで実現** すること。

### Q6: OT 1 回で MPC 全体ができるのか?

**いいえ、大量に必要**。Boolean 回路で AND ゲート数だけ OT を使う(GMW)。Yao's GC は入力ビット数だけ使う。1 関数評価で数百万 OT ということも普通。

### Q7: OT Extension で何が変わる?

**劇的に速くなる**。$\kappa$ 回の Base OT(公開鍵)だけで、$n$ 回の効果的 OT($n \gg \kappa$)を作れる。公開鍵演算のコストを $O(n)$ から $O(\kappa)$ に削減。Article 8 で詳述。

### Q8: 「Oblivious」という言葉の意味は?

**「気づかない」「意識しない」**の意。Sender が Receiver の選択を知らず、Receiver が選ばなかった秘密を知らない、**両方が「相手の情報に対して Oblivious」**という状態を表す。

### Q9: OT の量子耐性版は必要?

**長期的には必要**。量子コンピュータが実用化すると DDH や RSA が破れる。Lattice-based OT、code-based OT の研究が進んでいる。NIST Post-Quantum Cryptography 選定とも関連。

### Q10: 実装を試したい。最初の一歩は?

1. **Python で Naor-Pinkas**: PyCryptodome で楕円曲線演算を実装
2. **libOTe** (C++): プロ向け、性能重視
3. **MP-SPDZ**: 一度に全部触れる
4. **論文読み**: Chou-Orlandi 2015 が読みやすい

## 9. まとめ

### 本記事で学んだこと

- **OT (Oblivious Transfer)**: Sender の2秘密 $m_0, m_1$ のうち、Receiver が選択 $b$ の $m_b$ だけを受け取る。両方向に秘匿性
- **Kilian (1988)**: OT があれば任意の 2PC/MPC が作れる(**MPC Complete**)
- **Naor-Pinkas**: DDH 仮定ベースの効率的 OT。$g^x, C$ を使った選択秘匿性
- **Semi-Honest 安全性**: Receiver 腐敗は DDH で、Sender 腐敗は Perfect Simulation で示す
- **応用**: Yao's GC、GMW、PSI、ORAM、閾値暗号 — ほぼ全 dishonest-majority MPC の基盤
- **課題**: 公開鍵演算が重い。**大量実行が必要**で、Article 8 の OT Extension で解決

### 次の記事(Article 8)へ

次は **OT Extension**。Beaver (1996) が「OT は大量に実行すれば公開鍵演算はわずか $\kappa$ 回で済む」と示し、Ishai-Kilian-Nissim-Petrank (IKNP, 2003) が実用レベルに到達させた。現代の MPC システムのコアで、**1秒で数千万 OT** を生成できる。

- Beaver の観察(非効率な理論構成)
- IKNP の matrix tricks
- Correlation-robust hash の役割
- 実装ライブラリと性能

### 3行サマリ

- **OT = Sender の2秘密のうち Receiver が選んだ片方だけを、互いに相手の情報を知らずに届けるプリミティブ**
- **Kilian 1988: OT があれば任意の2者間MPCが構築可能(MPC Complete)**
- **Naor-Pinkas が実装の定番、ただし大量実行には Article 8 の OT Extension が必須**

---

## 参考文献

- Joe Kilian. *Founding Cryptography on Oblivious Transfer*. STOC 1988.
- Moni Naor, Benny Pinkas. *Efficient Oblivious Transfer Protocols*. SODA 2001.
- Mihir Bellare, Silvio Micali. *Non-Interactive Oblivious Transfer and Applications*. CRYPTO 1989.
- Tung Chou, Claudio Orlandi. *The Simplest Protocol for Oblivious Transfer*. LATINCRYPT 2015.
- Chris Peikert, Vinod Vaikuntanathan, Brent Waters. *A Framework for Efficient and Composable Oblivious Transfer*. CRYPTO 2008.
- Russell Impagliazzo, Steven Rudich. *Limits on the Provable Consequences of One-way Permutations*. STOC 1989.
