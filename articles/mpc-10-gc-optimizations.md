---
title: "[MPC入門シリーズ 10/15] GC の最適化 — FreeXOR、Half-Gates、そして帯域を削る技"
emoji: "🔧"
type: "tech"
topics:
  - mpc
  - cryptography
  - garbledcircuits
  - freexor
  - halfgates
published: false
---

**日付**: 2026年4月24日
**学習内容**: Article 9 で基本形の Yao's GC を見た。素朴には 1 ゲート 4 ciphertexts($4 \kappa$ bit)だが、**20 年以上の最適化**で 1 AND ゲート 2 ciphertexts、XOR ゲート 0 ciphertexts まで削減されている。本記事ではその道のりを辿る: (1) **Garbled Row Reduction (GRR3, GRR2)**、(2) **FreeXOR** — XOR を無料にする発想、(3) **Half-Gates** — 2016 年の決定版、(4) **Fixed-key AES Garbling** — AES-NI を活用した高速実装、(5) **FleXOR** と **Privacy-Free GC** などの派生、(6) 最適化を組み合わせた**現代の実装**の概観。これで 1 AES 暗号化を 200 KB 以下、数 ms で完了する水準に到達する。

## 0. 本記事の位置づけ

Yao's GC の素朴版(Article 9)は、1 ゲート 4 ciphertexts($4 \kappa = 512$ bit)で済ませる。これでも十分すごいが、帯域が通信と計算のボトルネック。10 万ゲートの回路で既に 64 MB の通信が発生する。

本記事で扱う最適化群は、**ほぼ同じ安全性**を保ちながら、1 AND = $2\kappa$ bit、XOR = 0 bit まで削減する。20 年以上の研究の蓄積で、AES 暗号化や SHA256 のような「実用的な回路」が**数 ms で MPC 評価可能**になった。

本記事の構成:

- **第1章**: Garbled Row Reduction (GRR3, GRR2)
- **第2章**: FreeXOR
- **第3章**: Half-Gates
- **第4章**: Fixed-key AES Garbling
- **第5章**: Privacy-Free GC と FleXOR
- **第6章**: 最適化の総覧表
- **第7章**: 実装と性能
- **第8章**: Q&A

## 1. Garbled Row Reduction (GRR)

### 1.1 アイデア (Naor-Pinkas-Sumner 1999)

素朴な GC では 1 ゲートに 4 エントリが必要。**そのうち 1 つを「送らない」**ことができないか?

**キー観察**: エントリ $(0, 0)$ は常に $\mathsf{Enc}_{k_a^0, k_b^0}(k_c^?) = H(k_a^0 \| k_b^0) \oplus (k_c^?)$。**出力ラベルをわざと $H(k_a^0 \| k_b^0)$ に一致させれば、このエントリは 0 で、送る必要がない**。

### 1.2 GRR3

Generator が $k_c^?$(このゲートの (0,0) case に対応する出力ラベル)を**ランダムに選ぶ代わりに**、$H(k_a^0 \| k_b^0)$ をそのままラベルとして使う。

結果:

- エントリ (0,0): 送信不要(値は 0)
- エントリ (0,1), (1,0), (1,1): 通常通り送信

**1 ゲート 3 ciphertexts**($3 \kappa$ bit)に削減。GRR3(Garbled Row Reduction、3 行残す)。

### 1.3 GRR2

Pinkas-Schneider-Smart-Williams (2009) はさらに進んで、**多項式補間**で 4 エントリ中 2 つしか送らない方法を提案。

**Polynomial Interpolation 版**:
- 4 エントリを 2 次多項式の 4 点とみなす
- 2 点(ciphertexts)だけ送れば、Evaluator が他を補間で復元

**問題**: この手法は FreeXOR と併用不可。FreeXOR の方が効果が大きいので、GRR2 は**ほぼ使われない**。

**Half-Gates (2015)** が FreeXOR 互換の「1 ゲート 2 エントリ」を実現し、状況を変えた(第3章)。

## 2. FreeXOR — XOR を無料にする

### 2.1 問題意識

Boolean 回路では AND, OR, XOR, NOT が混在する。特に **XOR は線形** なので、「暗号化されていても線形性を使える」のでは?

### 2.2 アイデア (Kolesnikov-Schneider 2008)

全 wire の 0/1 ラベルを**特殊な関係**で構成する:

$$
k_w^1 = k_w^0 \oplus \Delta
$$

ここで $\Delta \in \{0,1\}^\kappa$ は **全 wire で共通のグローバルシフト**。Generator が最初にランダムに選び、以後固定。

### 2.3 XOR ゲートの garbling

XOR ゲート $c = a \oplus b$ を考える。**Generator はラベルを次のように定める**:

$$
k_c^0 = k_a^0 \oplus k_b^0
$$

これで:

$$
k_c^1 = k_c^0 \oplus \Delta = (k_a^0 \oplus k_b^0) \oplus \Delta
$$

任意の $(a, b)$ に対して:

$$
k_c^{a \oplus b} = k_a^a \oplus k_b^b
$$

が成立することが、$\Delta$ の線形性から確認できる。

**結果: XOR ゲートは garbled table 不要!** Evaluator は 2 つの入力 label を XOR するだけ。

- **通信**: 0 bit
- **計算**: 1 回の XOR

### 2.4 AND ゲートへの影響

AND は依然として 4 エントリ必要。しかし、**$\Delta$ を固定しているので、出力ラベルも $k_c^1 = k_c^0 \oplus \Delta$ の形で決まる**。garbling のランダム性は $k_c^0$ のみ。

### 2.5 セキュリティ

FreeXOR の安全性は **素朴な PRG 仮定では不十分**。**Correlation Robust Hash** が必要(Article 8 と同じ概念)。Choi-Katz-Kumaresan-Zhou (2012) が厳密な仮定を pin down。

実装では Random Oracle or fixed-key AES を RO として使う(heuristic だが安全)。

### 2.6 効果

AES 回路(約 6800 ゲート)のうち、**多くが XOR**。FreeXOR で **通信を 5〜10 倍削減**できる。Yao's GC の歴史で最大のブレークスルーの1つ。

## 3. Half-Gates — 2015 年の決定版

### 3.1 動機

FreeXOR は XOR を無料にしたが、**AND は 3 エントリ(GRR3)+ FreeXOR互換**が当時の最先端。Zahur-Rosulek-Evans (2015) が **2 エントリの AND ゲート** を FreeXOR 互換で実現。

### 3.2 Half-Gate の分解

AND ゲート $c = a \wedge b$ を、2 つの**半ゲート**に分解:

$$
a \wedge b = (a \wedge r) \oplus (a \wedge (b \oplus r))
$$

ここで $r$ はランダムビット。各項は「**片方の入力が既知**」な AND で、これを **half-gate** と呼ぶ。

具体的には:

1. **Generator-half-gate**: Generator が $r$ を知り、$a \wedge r$ を計算
2. **Evaluator-half-gate**: Evaluator が $(b \oplus r)$ を知り、$a \wedge (b \oplus r)$ を計算
3. 2つを XOR で合成 → FreeXOR で無料

### 3.3 1 エントリの Half-Gate

各 half-gate は **1 エントリ**で済むことが示せる(GRR3 の変形で)。結果、**1 AND = 2 ciphertexts**。

### 3.4 具体的な構成(概略)

$a$, $b$ の labels $k_a^0, k_a^1 = k_a^0 \oplus \Delta$ と $k_b^0, k_b^1 = k_b^0 \oplus \Delta$ を仮定。

Generator は $r$ として $k_b^0$ のポインタビット(point-and-permute で決まるランダムビット)を選ぶ。

**Half-gate 1** (Generator knows $r$):

- Evaluator が持つ $k_a^{v_a}$(pointer bit $p$)に対し、$H(k_a^{v_a})$ から $k^{v_a \wedge r}$ を導出
- 1 エントリ: $H(k_a^0) \oplus H(k_a^1) \oplus \Delta$(Generator が事前送信)

**Half-gate 2** (Evaluator knows $b \oplus r$):

- Evaluator が持つ $k_b^{v_b}$ から $k^{v_a \wedge (v_b \oplus r)}$ を導出
- 1 エントリ: $H(k_b^0) \oplus H(k_b^1) \oplus k_a^0$

両方の結果を XOR で合流(FreeXOR で無料)。

### 3.5 合計コスト

**1 AND ゲート**:
- 通信: 2 ciphertexts = $2\kappa$ bit
- 計算: Generator 4 回ハッシュ、Evaluator 2 回ハッシュ

**1 XOR ゲート**: 0 bit(FreeXOR)

Zahur-Rosulek-Evans (2015) は、これ以上の削減はある種の自然な制約の下で**不可能**であることも示した(最下限性)。

## 4. Fixed-key AES Garbling

### 4.1 背景

GC のガーブル関数 $H$ として、AES や SHA-256 を使う。しかし毎回ハッシュ計算すると、**鍵スケジュール(AES key schedule)のオーバーヘッド**が無視できない。

### 4.2 アイデア (Bellare-Hoang-Keelveedhi-Rogaway 2013)

**固定の AES 鍵 $K$** を使い、AES を「固定鍵のランダム置換」として使う:

$$
H(x) = \mathsf{AES}_K(x) \oplus x
$$

(Davies-Meyer 構成で擬似ランダムな出力)

### 4.3 Intel AES-NI

Intel CPU の **AES-NI 命令**(AES 専用ハードウェア)で、AES 暗号化が**数サイクル**で完了。Fixed-key にすればスケジュールが不要で、ほぼロスなし。

### 4.4 効果

**実装速度が数倍**。Garbled circuit の生成・評価のスループットを劇的に向上。

### 4.5 安全性の注意

Bellare et al. は **Ideal Cipher** として AES を扱う heuristic 仮定を置く。Guo-Katz-Wang-Yu (2020) が実用的な代替を提案し、近年の実装(EMP、swanky)はこれを採用。

## 5. Privacy-Free GC と FleXOR

### 5.1 Privacy-Free GC

**Frederiksen-Nielsen-Orlandi (2016)**: Zero-Knowledge Proofs(ZK)では **Evaluator がすでに入力を知っている**(Prover の入力を検証するだけ)。この場合、**ラベルに秘匿性は不要**。1 AND ゲート 1 ciphertext まで削減可能。

MPC for ZK は近年の研究トピック。ZKP プロトコルの高速化に使う。

### 5.2 FleXOR (Kolesnikov-Mohassel-Rosulek 2014)

FreeXOR は全 wire で同じ $\Delta$ を使う。**FleXOR** は wire ごとに異なる $\Delta$ を許す代わりに、回路最適化でバランスを取る。

FreeXOR で「共通 $\Delta$」が安全性の仮定を重くしている部分を、**部分的に回避**する。実装は複雑で、Half-Gates に大枠で抜かれた感がある。

## 6. 最適化の総覧

| 最適化 | 年 | AND cost | XOR cost | 互換 |
|---|---|---|---|---|
| 素朴 | 1986 | $4\kappa$ | $4\kappa$ | - |
| Point-and-Permute | 1990 | $4\kappa$ | $4\kappa$ | 必須 |
| GRR3 | 1999 | $3\kappa$ | $3\kappa$ | P&P |
| GRR2 (interp) | 2009 | $2\kappa$ | $2\kappa$ | FreeXOR 非互換 |
| FreeXOR | 2008 | $3\kappa$ | $0$ | P&P + GRR3 |
| FleXOR | 2014 | $2\kappa$ | $0\sim 2\kappa$ | 部分的 |
| **Half-Gates** | 2015 | $2\kappa$ | $0$ | FreeXOR + P&P |

**現代の標準**: **Half-Gates + FreeXOR + Point-and-Permute + Fixed-key AES**。

## 7. 実装と性能

### 7.1 ベンチマーク(2024 年、LAN)

AES-128 暗号化回路を 2PC で実行:

- **素朴 (1986)**: $\sim 500$ ms、数 MB 通信
- **Half-Gates + Fixed-key AES**: $\sim 2$ ms、$\sim 150$ KB 通信

**200 倍以上の改善**。

### 7.2 ライブラリ

- **EMP-toolkit**: C++、Half-Gates + Fixed-key AES
- **MP-SPDZ**: Python-like、複数の garbling スキーム選択可
- **swanky** (Galois): Rust、Half-Gates

### 7.3 ハードウェア活用

- **AES-NI**: Intel、AMD で標準搭載、10 倍以上高速化
- **AVX-512**: バッチ処理で並列化
- **GPU**: 大規模 MPC で実験的、数十倍加速

## 8. Q&A

### Q1: XOR が無料? 本当に?

**本当**。FreeXOR で garbled table を送らず、Evaluator は 2 つのラベルを XOR するだけ。ただし **Random Oracle か correlation robust hash** の仮定が必要。

### Q2: Half-Gates で 1 ciphertext はできないの?

**不可能**(Zahur-Rosulek-Evans 2015 の下限)。自然な条件下で、AND には最低 2 ciphertexts 必要。

### Q3: 3-入力ゲートや他のゲートは?

**Ball-Malkin-Rosulek (2016)** など研究あり。高 fan-in ゲートは表現力が高いが、garbled table が指数的に増える。実用は 2-入力限定。

### Q4: 並列化は?

**できる**。Generator は独立にゲートを garbling でき、Evaluator も入力が揃ったゲートから並列評価可能。EMP などで実装済。

### Q5: 低帯域ネットワーク(WAN)での性能は?

**通信量が支配的**。Half-Gates で $2\kappa$ bit/AND なので、WAN でも 100 Mbps あれば十分速い。1 Mbps でも AES 暗号化は数百 ms で終わる。

### Q6: Malicious-secure GC にもこれらの最適化は使える?

**ほぼ使える**。ただし Cut-and-Choose では GC を複数作るので、最適化効果が $\times \rho$($\rho$ = レプリケーション係数、典型 40)で打ち消される。Authenticated Garbling (Article 13) では大部分が使える。

### Q7: Quantum computer で破れる?

**AES は Grover で $\sqrt{N}$ 倍効率化**されるが、$\kappa = 256$ なら 128 bit セキュリティが保てる。Post-Quantum GC は研究中。

### Q8: 実装で気をつけること?

- **乱数生成の質**: 暗号学的に安全な RNG
- **メモリ使用量**: 巨大回路で GB 単位、ストリーミング実装が良い
- **AES-NI の確認**: CPU が対応しているか(`cpuid`)
- **Correct hash H**: fixed-key AES + Davies-Meyer

### Q9: どれくらいの回路サイズまで実用?

**2024 年時点**:
- 小〜中(〜$10^5$ gates): 即座に完了(< 1 秒)
- 大($10^6 \sim 10^7$): 数分以内
- 超大($10^9$): 数時間。最適化・並列化で数分に短縮可能

Article 14 で実装詳細。

### Q10: Arithmetic Garbled Circuit はある?

**ある**。Ball-Malkin-Rosulek (2016), Applebaum-Ishai-Kushilevitz (2011)。$\mathbb{F}_p$ 上の算術ゲートを garbling するが、Boolean 版より効率悪い。用途は ZK や arithmetic-heavy な問題に限る。

## 9. まとめ

### 本記事で学んだこと

- **Garbled Row Reduction (GRR3)**: 4 エントリ → 3 エントリ、Generator の自由度でゼロ項を作る
- **FreeXOR (2008)**: 全 wire で共通 $\Delta$ を使い、XOR ゲートを通信ゼロに
- **Half-Gates (2015)**: AND を 2 half-gates に分解し、1 AND = $2\kappa$ bit に圧縮
- **Fixed-key AES Garbling**: AES-NI でガーブル関数 H を超高速化
- 現代の標準: **Half-Gates + FreeXOR + Point-and-Permute + Fixed-key AES**
- **200 倍以上の性能改善**を実現(1986 年の素朴版比)

### 次の記事(Article 11)へ

次は **GMW プロトコル**。Yao's GC が 2 者に特化しているのに対し、**GMW は $n$ 者に自然に拡張**できる秘密分散ベース MPC。

- Boolean 回路の XOR 秘密分散
- AND ゲートでの 1-out-of-4 OT
- 回路深さに比例するラウンド数
- Beaver triple との関係(Article 12 への布石)

### 3行サマリ

- **最適化の到達点: 1 AND = $2\kappa$ bit (Half-Gates)、1 XOR = 0 bit (FreeXOR)**
- **Random Oracle / correlation robust hash 仮定が必要だが、実装は fixed-key AES で超高速**
- **素朴版比で 200 倍以上の性能改善 — AES 暗号化の 2PC が数 ms で完了する水準**

---

## 参考文献

- Moni Naor, Benny Pinkas, Reuban Sumner. *Privacy Preserving Auctions and Mechanism Design*. EC 1999. (GRR3)
- Vladimir Kolesnikov, Thomas Schneider. *Improved Garbled Circuit: Free XOR Gates and Applications*. ICALP 2008. (FreeXOR)
- Benny Pinkas, Thomas Schneider, Nigel Smart, Stephen Williams. *Secure Two-Party Computation is Practical*. ASIACRYPT 2009. (GRR2)
- Vladimir Kolesnikov, Payman Mohassel, Mike Rosulek. *FleXOR: Flexible Garbling for XOR Gates*. CRYPTO 2014.
- Samee Zahur, Mike Rosulek, David Evans. *Two Halves Make a Whole: Reducing Data Transfer in Garbled Circuits Using Half Gates*. EUROCRYPT 2015. (Half-Gates)
- Mihir Bellare, Viet Tung Hoang, Sriram Keelveedhi, Phillip Rogaway. *Efficient Garbling from a Fixed-Key Blockcipher*. IEEE S&P 2013.
- Seung Geol Choi, Jonathan Katz, Ranjit Kumaresan, Hong-Sheng Zhou. *On the Security of the "Free-XOR" Technique*. TCC 2012.
- Chun Guo, Jonathan Katz, Xiao Wang, Yu Yu. *Efficient and Secure Multiparty Computation from Fixed-Key Block Ciphers*. IEEE S&P 2020.
