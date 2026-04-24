---
title: "[完全準同型暗号入門シリーズ 10/15] ブートストラッピング — Gentryの決定打"
emoji: "♻️"
type: "tech"
topics:
  - fhe
  - bootstrapping
  - gentry
  - circular
  - refresh
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事は完全準同型暗号の心臓部 **ブートストラッピング (bootstrapping)** を扱う。Gentry が 2009 年に示した「**暗号化された状態で復号回路を評価すればノイズがリセットできる**」というアイデアは、「**Somewhat HE → Fully HE**」を渡した決定的橋だった。本記事では **(1) ブートストラッピングの原理**、**(2) 「なぜ暗号の中で暗号を復号できるのか」**、**(3) Gentry のオリジナル構成**、**(4) Circular security 仮定**、**(5) BGV / BFV / CKKS / TFHE それぞれの bootstrap**、**(6) 実測性能**、**(7) TFHE の gate bootstrapping**、**(8) Programmable bootstrapping** を扱う。ここを理解すれば、FHE の「完全性」がなぜ可能かの最後のピースが埋まる。

## 0. 本記事の位置づけ

Article 9 で Leveled FHE により **乗算深さ $L$ まで計算可能** と分かった。しかし:

- **$L$ を使い切ったらどうする？**
- **事前に深さが見積もれない計算は？**
- **無制限回数の計算を実現したい**

これらに答えるのが **Bootstrapping**。2009 年の Gentry の博士論文でこの仕組みが示されたことで、「完全」準同型暗号が現実になった。

アイデアは天才的だが、**原理自体はシンプル**:

> **「復号は単なる計算。計算なら FHE で暗号化されたまま実行できる」**

構成:

- **第1章**: 問題設定 — ノイズ上限後の世界
- **第2章**: Bootstrapping の原理
- **第3章**: 暗号の中で暗号を復号する
- **第4章**: Circular security
- **第5章**: Gentry のオリジナル構成
- **第6章**: BGV/BFV/CKKS の bootstrap
- **第7章**: TFHE の gate bootstrapping
- **第8章**: Programmable bootstrapping
- **第9章**: Q&A とまとめ

## 1. 問題設定 — ノイズ上限後の世界

### 1.1 Leveled FHE の限界

モジュラスチェーン $q_0 > q_1 > \cdots > q_L$ を使い切ると:

- 暗号文が $q_L$（最小）上にある
- 次の乗算をすると **ノイズが $q_L/4$ を越え** 復号失敗

選択肢:
- **諦める**（深い計算はできない）
- **$L$ を最初から大きく**（$q_0$ が巨大化、全体が遅い）
- **ノイズを復元**（= Bootstrapping）

### 1.2 ノイズを復元するとは

暗号文 $\text{ct}$ の平文を $m$ とする。欲しいのは:

- 同じ平文 $m$ を暗号化した **新しい暗号文** $\text{ct}_{\text{new}}$
- そのノイズが **小さい**（新鮮な暗号文レベル）
- **平文を明かさず** に実現

もし「復号してから再暗号化」できれば:

1. $m = \text{Dec}(\text{ct}, sk)$
2. $\text{ct}_{\text{new}} = \text{Enc}(m, pk)$

しかしクライアントに一度送り返すのは本末転倒。**サーバだけで実現したい**。

## 2. Bootstrapping の原理

### 2.1 Gentry の天啓

ノイズの大きい暗号文 $\text{ct}$ に対して、サーバで:

1. **復号アルゴリズム $\text{Dec}$ を関数として見る**
2. その関数を **FHE で暗号化された状態で評価**

復号関数 $\text{Dec}(c, sk)$ への入力:
- $c$: 復号対象の暗号文（これは公開情報、サーバが持つ）
- $sk$: 秘密鍵 → **これは暗号化されている必要がある**

### 2.2 評価鍵: Enc(sk)

鍵生成時に **秘密鍵 $sk$ を秘密鍵自身で暗号化** した `evk` を公開する:

$$
\text{evk} = \text{Enc}_{sk}(sk)
$$

これを **bootstrap key** とも呼ぶ。これにより、復号関数を暗号の中で実行できる。

### 2.3 Bootstrap アルゴリズム

```
Bootstrap(ct_old, evk, pk):
    # ct_old: ノイズが大きい暗号文 (平文 m を暗号化)
    # evk = Enc(sk)
    # 復号関数 D(c, sk) を FHE で評価
    ct_new = Eval(D, ct_old, evk)
    return ct_new   # 新しい暗号文 (平文 m、ノイズ小)
```

### 2.4 なぜノイズが小さくなるか

$\text{Eval}(D, \text{ct}_{\text{old}}, \text{evk})$ の出力は:

- 平文: $D(\text{ct}_{\text{old}}, sk) = m$ ← 正しい復号結果
- ノイズ: $\text{Eval}$ による **新しく生じるノイズ**

この「新しく生じるノイズ」は **$\text{evk}$ と Eval の実装** に依存する。典型的には $O(\sqrt{n}\sigma)$ 程度で、**$\text{ct}_{\text{old}}$ のノイズより遥かに小さい**。

結果、新しい暗号文 $\text{ct}_{\text{new}}$ は **平文 $m$** を暗号化しており、**ノイズが小さく**、追加で乗算を積める。

## 3. 暗号の中で暗号を復号する

### 3.1 条件: 復号回路が浅い

Eval が機能するには、**復号関数 $D$ の回路深さが、現在のFHEスキームが評価できる深さより浅い** 必要がある:

$$
\text{depth}(D) < L(\text{scheme})
$$

これを **bootstrappable** な条件と呼ぶ。

### 3.2 復号の浅化

Regev スキームの復号は $v = c_0 + c_1 s$ の線形演算 + 閾値判定。

- 線形演算: **1 つの加算と 1 つの乗算** で済む（深さ $O(1)$）
- 閾値判定 (rounding): **ビット分解と比較** が必要 → これが深い

この「閾値判定」を浅くすることが bootstrapping 効率の鍵。

### 3.3 工夫: squashed decryption

Gentry のオリジナル構成では、**squashed decryption** という工夫で復号回路を浅くした:

1. 秘密鍵を **スパースな部分和** に書き換える
2. 復号が **少数個の項の和** に簡約
3. 深さが $O(\log \log q)$ まで浅くなる

### 3.4 復号を暗号化の中で「シミュレート」

復号アルゴリズムを形式的に:

```
Dec(c, s):
    v = c_0 + poly_mul(c_1, s)     # 乗算 1 回
    v_bits = bit_decompose(v)       # 複数の加算
    m = threshold(v_bits)            # 閾値
    return m
```

各ステップを **評価鍵 $\text{evk} = \text{Enc}(s)$ を使って** 暗号領域で実行する。

## 4. Circular Security

### 4.1 安全性の懸念

**秘密鍵 $sk$ をその鍵自身で暗号化する** ことは、普通の IND-CPA 安全性では保証されない。

- IND-CPA: 任意のメッセージを暗号化しても安全
- しかし $\text{Enc}_{sk}(sk)$ は「暗号化対象が鍵そのもの」という特殊ケース

一般に $\text{Enc}_{sk}(sk)$ が意味的に安全であることは **追加の仮定**（**circular security**）が必要。

### 4.2 Circular security 仮定

**KDM (Key-Dependent Message) 安全性** の一種:

> **$\text{Enc}_{sk}(f(sk))$ が意味的に安全** （$f$ は任意の多項式時間関数）

格子ベース暗号ではこの仮定が成立すると信じられている。しかし **証明されていない**。

### 4.3 証明された回避策

Circular security を避ける構成もある:

- **鍵の階層化**: $sk_0$ で $sk_1$ を暗号化、$sk_1$ で $sk_2$ を暗号化...
- **有限回の bootstrapping** を前提にする
- **Gentry-Sahai-Waters (GSW)**: circular security を仮定しない構成

実用ライブラリでは circular security を仮定している。

## 5. Gentry のオリジナル構成

### 5.1 Ideal lattice ベース

Gentry 2009 は **ideal lattice** ($\mathbb{Z}[X]/(X^n+1)$ のイデアル) を使う。

大まかな構造:

1. **Somewhat HE**: ideal lattice 上で基本の加算・乗算を定義
2. **Squashed scheme**: 復号を浅くする変換
3. **Bootstrapping**: 暗号化された秘密鍵で復号を評価

### 5.2 パラメータ

オリジナル構成では:
- $n = 2^{15} = 32768$
- Bootstrap 1 回 = **数分〜数十分**
- 実用にならないが、**原理を示した** ことが革命的

### 5.3 歴史的意義

Gentry の構成は実用には遅かったが、「**FHE が存在する**」ことを示した最初の構成。以降の改良スキーム（BGV, BFV, CKKS, TFHE）もこの枠組みに従っている。

## 6. BGV/BFV/CKKS の bootstrap

### 6.1 共通のアプローチ

これらのスキームの bootstrap は:

1. モジュラス $q_L$ の暗号文から出発
2. $q_L$ を **大きい $q_0$ にブローアップ** する
3. $q_0$ 上で復号回路を評価
4. 出力は **ノイズが小さい $q_0$ 暗号文**
5. 必要なら mod switch で所望のレベルに

### 6.2 性能

- **Halevi-Shoup (2015)**: BGV で 6 分
- **FHE over the Torus (TFHE, 2016)**: **0.013 秒 / gate**
- **CKKS bootstrap (2018)**: 数十秒〜数分（バッチで償却すると許容）

BGV/BFV/CKKS の bootstrap は依然 **重い**（1 回秒オーダー）。TFHE が圧倒的に速い。

### 6.3 BGV/BFV の bootstrap 詳細

1. **SlotToCoeff**: SIMD スロット表現を係数表現に
2. **Modulus raising**: $q_L \to q_0$
3. **Evaluate decryption circuit**: $\bmod q_0$ 上で復号関数を評価
4. **CoeffToSlot**: 係数表現を SIMD スロットに戻す

各ステップが **非常に重い多項式演算**。

### 6.4 CKKS の bootstrap

CKKS の bootstrap は BGV/BFV と類似だが:

- 実数演算なので丸め誤差の扱いが難しい
- **sine/cosine 近似** で mod operation を多項式化

Han-Ki-Kim (2018) 以降、数秒で 1 bootstrap が可能に。

## 7. TFHE の gate bootstrapping

### 7.1 TFHE の革新

Chillotti-Gama-Georgieva-Izabachène (CGGI, 2016-2020) の **TFHE** は、bootstrap を **ゲートレベル** で実行することで、**1 gate あたり 13 ms** まで高速化した。

### 7.2 gate bootstrapping の原理

TFHE の平文は **1 ビット** (または小さい整数)。

1. Boolean gate (AND, OR, NOT, XOR...) を暗号文上で実行
2. **毎ゲート後に bootstrapping** でノイズをリセット

これにより、**任意の深さの論理回路** が実行できる。乗算深さを気にしなくてよい。

### 7.3 blind rotation

TFHE bootstrap の核心アルゴリズム **blind rotation**:

1. 入力 LWE 暗号文の復号値 $\mu$ を見ずに、$\mu$ だけ「多項式を回転」
2. 回転後の多項式の先頭係数 = 所望の論理出力
3. 全体を RGSW 暗号で encapsulate

これにより、**復号せずに LWE 暗号文の値に依存した計算** ができる。

### 7.4 性能比較

| スキーム | Bootstrap 時間 | 演算粒度 |
|---|---|---|
| Gentry 2009 | 数分 | 全体 |
| BGV/BFV | 数秒 | バッチ SIMD |
| CKKS | 数秒〜数十秒 | バッチ SIMD |
| **TFHE** | **13 ms** | 1 gate |

TFHE が **桁違いに速い** のは、bootstrap を軽量な 1-gate にしたため。ただし、SIMD バッチ性能では BGV/CKKS が圧倒的に上。**用途による使い分け** が重要。

## 8. Programmable bootstrapping

### 8.1 アイデア

TFHE の bootstrap は、内部で「テーブルルックアップ」を行っている。**このテーブルを自由に変えられれば、任意の 1 変数関数を bootstrap と同じコストで評価できる**。

これが **programmable bootstrapping (PBS)**:

$$
\text{PBS}_f(\text{Enc}(x)) = \text{Enc}(f(x))
$$

### 8.2 応用

- **非線形活性化**: ReLU、sigmoid、tanh などが 1 PBS で実行可能
- **比較演算**: $x > 0$ の判定
- **任意の LUT**: カスタム関数

ReLU を多項式近似する必要がなくなり、**精度が上がる + 速い**。

### 8.3 CKKS への移植

最近 (2024-2025) の研究で、CKKS にも PBS 風の仕組みを組み込む試みが進んでいる。「**CKKS の高い SIMD 並列性** + **TFHE の軽量 bootstrap**」のハイブリッド。

## 9. Q&A

### Q1: Bootstrapping がなくても FHE と呼べる？

**呼べない**。「完全」FHE の定義が「無制限回数の計算」なので、bootstrapping が存在することが FHE の本質。

ただし実用では Leveled FHE で事足りるケースも多い。

### Q2: Bootstrapping は毎回必要？

**必要ない**。Leveled で収まる計算は bootstrap 不要。深い計算や動的計算で必要になる。

### Q3: Bootstrapping は安全性を下げる？

**Circular security を仮定する必要がある**。これは証明されていない仮定だが、実用上問題視されていない。

### Q4: Bootstrap 中にノイズはどう増える？

**入力のノイズには依存しない**（だからこそリセットできる）が、**bootstrap の実装自体**が一定のノイズを出力に追加する。このノイズ量が「**bootstrap 後のノイズレベル**」で、これが新鮮な暗号文のノイズより小さければ OK。

### Q5: TFHE の 13ms は実用的？

**用途次第**。1 gate 13ms なら、1000 gate の回路で 13 秒。単純な判定や小さい計算は実用的だが、ディープラーニングのような大規模計算は厳しい。

### Q6: Bootstrap のハードウェア加速はある？

**進行中**。FPGA/ASIC 実装で数倍〜数十倍の加速。Intel HEXL (AVX-512) はソフトウェアレベル。DARPA の DPRIVE プロジェクトで専用 ASIC も開発中。

## 10. まとめ

### 本記事で学んだこと

- **Bootstrapping**: 暗号化された状態で **復号関数を評価** してノイズをリセットする魔法
- **評価鍵 $\text{evk} = \text{Enc}(sk)$**: bootstrap 実行に必要。**Circular security 仮定** が必要
- **条件**: 復号回路の深さ < スキームの評価能力
- **Gentry 2009**: 原理を示したが遅い（数分）
- **BGV/BFV/CKKS**: 改良版の bootstrap（数秒）
- **TFHE gate bootstrap**: **13 ms / gate** と圧倒的に速い
- **Programmable bootstrapping**: 任意の 1 変数関数を bootstrap と同コストで評価

### 次の記事（Article 11）へ

次の記事からは、実用 FHE の 3 大スキームを順に深掘りする。まずは **BGV と BFV** — 整数演算に最適化された第 2 世代スキーム。SIMD バッチング (CRT packing) と、そこから自然に派生する **スロット回転 (rotation)** の仕組みを扱う。

### 3行サマリ

- **Bootstrapping** = 暗号化されたまま **復号関数を評価** してノイズを小さくする操作
- 「暗号の中で暗号を復号する」成立条件は **復号回路が浅いこと** + **circular security**
- TFHE の **gate bootstrapping** は 13ms と圧倒的に速い。PBS なら任意 1 変数関数が同コスト

---

## 参考文献

- Craig Gentry. *Fully Homomorphic Encryption Using Ideal Lattices.* STOC 2009.
- Gentry. *Computing Arbitrary Functions of Encrypted Data.* Communications of the ACM, 2010.
- Chillotti, Gama, Georgieva, Izabachène. *TFHE: Fast Fully Homomorphic Encryption over the Torus.* Journal of Cryptology, 2020.
- Ducas, Micciancio. *FHEW: Bootstrapping Homomorphic Encryption in Less Than a Second.* EUROCRYPT 2015.
- Han, Ki, Kim. *Better Bootstrapping for Approximate Homomorphic Encryption.* CT-RSA 2020.
