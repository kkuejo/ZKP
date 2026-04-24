---
title: "[完全準同型暗号入門シリーズ 11/15] BGV と BFV — 整数演算のための第2世代FHE"
emoji: "🔢"
type: "tech"
topics:
  - fhe
  - bgv
  - bfv
  - simd
  - batching
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事から実用 FHE 3 大スキームを順に深掘りする。まずは **BGV (Brakerski-Gentry-Vaikuntanathan, 2011)** と **BFV (Fan-Vercauteren, 2012)**。どちらも Ring-LWE ベースで **整数演算** に最適化され、Microsoft SEAL、OpenFHE、HElib などの主要ライブラリが実装する。本記事では **(1) BGV の構造**、**(2) BFV の構造**、**(3) BGV vs BFV の違い**、**(4) SIMD バッチング (CRT packing)**、**(5) スロット回転 (rotation)**、**(6) ガロア自動化と key switch**、**(7) パラメータ選択の実際**、**(8) Microsoft SEAL での API 例** を扱う。BGV/BFV は「整数の行列・ベクトル演算」を暗号化したまま実行する道具として、現在も最も広く使われている。

## 0. 本記事の位置づけ

Article 7-10 で FHE の一般原理（加算、乗算、再線形化、noise、bootstrap）を学んだ。本記事から **具体的なスキーム** に入る。

実用 FHE は主に 3 種類:

1. **BGV / BFV** (本記事): 整数演算、SIMD バッチング
2. **CKKS** (Article 12): 近似実数演算、ML 向け
3. **TFHE** (Article 13): ビット演算、高速 bootstrap

いずれも Ring-LWE ベースで、暗号文の形も類似しているが、**平文空間の定義** と **ノイズ管理の仕方** が違う。

BGV と BFV は兄弟スキーム。論文が 1 年違いで発表され、どちらも「整数演算 FHE」。細かい違いはあるが、実用上はほぼ互換的。

構成:

- **第1章**: BGV の構造
- **第2章**: BFV の構造
- **第3章**: BGV vs BFV
- **第4章**: SIMD バッチング
- **第5章**: スロット回転
- **第6章**: ガロア自動化
- **第7章**: パラメータ
- **第8章**: SEAL API 例
- **第9章**: Q&A とまとめ

## 1. BGV の構造

### 1.1 パラメータ

- **次数** $n$: 2 のべき（例: $n = 4096, 8192$）
- **暗号文モジュラス** $q$: 数個の NTT-friendly 素数の積
- **平文モジュラス** $t$: 通常 $t \ll q$ の素数（例: $t = 65537 = 2^{16} + 1$）
- **ノイズ分布** $\chi$: ガウス、$\sigma \approx 3.19$

### 1.2 鍵生成

```
Secret key: s ← R_q (スパース、係数 ∈ {-1, 0, 1})
Public key: (a, b) where a ← R_q (一様), b = -a·s + t·e (e ← χ)
Relin key:  rlk = Enc_s(s^2) (gadget decomposition付)
```

**注意**: BGV では公開鍵が $b = -a \cdot s + t \cdot e$ と、ノイズに $t$ が掛かる。

### 1.3 暗号化

```
平文 m ∈ R_t
u, e_0, e_1 ← χ
c_0 = b · u + t · e_0 + m  (mod q)
c_1 = a · u + t · e_1      (mod q)
ct = (c_0, c_1)
```

### 1.4 復号

```
c_0 + c_1 · s = (-a·s + t·e) · u + t·e_0 + m + (a·u + t·e_1) · s
              = t · (e·u + e_0 + e_1·s) + m
              = t · (noise) + m   (mod q)
```

**平文 $m$ は $\bmod t$ の最下位、ノイズは $t$ の倍数** にある。復号は $\bmod q$ したあと $\bmod t$:

```
v = c_0 + c_1·s (mod q)
v_signed = (v + q/2) mod q - q/2
m = v_signed mod t
```

### 1.5 BGV の加算・乗算

- **加算**: 成分ごとの $+ \bmod q$
- **乗算**: Article 8 の素朴な乗算 + 再線形化 + **mod switch**
- **Mod switch**: $q \to q' = q / p_i$ でノイズを管理

BGV は **平文が $\bmod q$ の最下位ビット側** にあるため、mod switch の扱いが直感的。

## 2. BFV の構造

### 2.1 パラメータ

BGV と基本的に同じ。$n, q, t, \chi$。

### 2.2 鍵生成

BFV では平文空間が **「最上位ビット側」** にある:

```
Secret key: s ← R_q
Public key: (a, b = -a·s + e)   ← ノイズに t は掛からない
```

### 2.3 暗号化

```
平文 m ∈ R_t
Δ = ⌊q/t⌋   (スケーリング係数)
u, e_0, e_1 ← χ
c_0 = b · u + e_0 + Δ · m    (mod q)
c_1 = a · u + e_1              (mod q)
```

**$\Delta \cdot m$ が暗号文の「最上位ビット側」を占める**。

### 2.4 復号

```
c_0 + c_1 · s = Δ · m + (small noise)   (mod q)
m = round(t · v / q) mod t
```

$v = c_0 + c_1 \cdot s$ を $t/q$ 倍して四捨五入。

### 2.5 BFV の乗算

乗算は特殊:

$$
(c_0 + c_1 s)(c_0' + c_1' s) = \Delta^2 m m' + \Delta \cdot (\text{noise}) + (\text{noise})^2
$$

両辺を $\Delta$ で割る必要があるが、除算は不可能。代わりに **$t/q$ を掛けて四捨五入**:

$$
\text{round}\left(\frac{t}{q} \cdot (\text{素朴乗算})\right)
$$

これで $\Delta m m' + (\text{smaller noise})$ の暗号文。

### 2.6 BFV の mod switch は不要？

**原則不要**。BFV はスケーリングで自動的にノイズを管理する。内部的には「noise を数えて残り budget を管理」する。

一部の実装（SEAL 新版）では BFV でもモジュラスチェーンを使う。

## 3. BGV vs BFV

### 3.1 数学的な違い

| 項目 | BGV | BFV |
|---|---|---|
| 平文位置 | 最下位 ($\bmod t$) | 最上位 ($\Delta m$) |
| ノイズ | $t \cdot \text{noise}$ | $\text{noise}$ |
| 乗算後処理 | mod switch | scaling |
| Modulus chain | 明示的に管理 | 内部管理 |

### 3.2 速度

**ほぼ同じ**。実装上の微妙な違いで、ワークロードによって多少変わる。

- **BGV が速い場合**: 平文モジュラス $t$ が大きいとき、mod switch を頻繁に使える
- **BFV が速い場合**: 単純な構造で実装オーバーヘッドが少ない

### 3.3 ノイズの扱い

BGV: ノイズは **$t$ の倍数** として分離されており、mod $t$ で完全に消える。計算が「整数環 $\mathbb{Z}_t$ の上で正確」

BFV: ノイズは **平文スケール $\Delta$ の「下側」** にあり、四捨五入で除去。

### 3.4 使い分け

多くの実装では両方提供し、ユーザーが選べる:

- **SEAL**: BFV がデフォルト、BGV もサポート
- **OpenFHE**: 両方サポート
- **HElib**: BGV 中心

**実用では BFV がやや多い**（SEAL が業界事実上の標準のため）。

## 4. SIMD バッチング (CRT packing)

### 4.1 並列化の魔法

BGV/BFV の **最大の武器** は SIMD バッチング。1 つの暗号文の中に **$n$ 個の独立な平文スロット** を詰め込める。

### 4.2 原理: 中国剰余定理

平文多項式 $m(X) \in R_t = \mathbb{Z}_t[X]/(X^n+1)$ を考える。

**$X^n + 1 = \prod_{i} f_i(X)$** という既約分解を $\bmod t$ で行う。$t$ を適切に選ぶと ($t \equiv 1 \bmod 2n$)、$X^n + 1$ は **$n$ 個の 1 次因子に分解**:

$$
X^n + 1 = \prod_{i=1}^{n} (X - \omega_i) \pmod t
$$

中国剰余定理 (CRT) により:

$$
R_t \cong \mathbb{Z}_t^n \quad \text{via} \quad m(X) \mapsto (m(\omega_1), m(\omega_2), \ldots, m(\omega_n))
$$

つまり **1 つの多項式 = $n$ 個の独立なスロット**。

### 4.3 Encode/Decode

```
Encode: [a_1, a_2, ..., a_n] (vector ∈ Z_t^n) 
        ↔ m(X) ∈ R_t   (多項式)

# encoding: CRT の逆変換 (IFFT-like)
# decoding: 多項式を各 ω_i で評価 (FFT-like)
```

これを **inverse NTT / NTT** で高速実行。

### 4.4 SIMD 演算

暗号化されたベクトル $[a_1, \ldots, a_n]$ と $[b_1, \ldots, b_n]$ に対して:

```
ct_a ⊕ ct_b = Enc([a_1 + b_1, a_2 + b_2, ..., a_n + b_n])   # 成分ごと加算
ct_a ⊗ ct_b = Enc([a_1 · b_1, a_2 · b_2, ..., a_n · b_n])   # 成分ごと乗算
```

1 回の暗号文演算で **$n$ 個の SIMD 演算** が実行できる。

### 4.5 具体例: 内積

2 つのベクトル $\vec{a}, \vec{b} \in \mathbb{Z}_t^n$ の内積 $\sum a_i b_i$ を計算する:

```
ct_a = Enc([a_1, a_2, ..., a_n])
ct_b = Enc([b_1, b_2, ..., b_n])

ct_prod = ct_a ⊗ ct_b   # Enc([a_1 b_1, a_2 b_2, ..., a_n b_n])

# 全スロットの和を取る (rotation + add)
ct_sum = ct_prod
for shift in [1, 2, 4, ..., n/2]:
    ct_sum = ct_sum ⊕ rotate(ct_sum, shift)

# 最終的に ct_sum の全スロットが sum(a_i * b_i) になる
```

$n = 4096$ で、**1 回の乗算 + log(n) 回のシフト加算 ≈ 13 回** で **4096 要素の内積**が計算できる。これは行列計算に絶大な威力。

## 5. スロット回転 (Rotation)

### 5.1 動機

SIMD バッチングで重要な演算:

- **Rotation**: $[a_1, a_2, \ldots, a_n] \to [a_2, a_3, \ldots, a_n, a_1]$
- **Sum of all slots**: 全スロットの和を取る（内積計算に必要）

これらをどう実現するか。

### 5.2 回転の仕組み

多項式 $m(X)$ の $X \to X^k$ の代入（$k$ は gcd(k, 2n) = 1 の奇数）は、スロット表現でどう見えるか:

$$
m(X^k) \leftrightarrow \text{permute}([m(\omega_1), \ldots, m(\omega_n)])
$$

つまり **$X \to X^k$ がスロットの置換**に対応する。

### 5.3 ガロア自動化 (Galois Automorphism)

$X \to X^k$ は **円分体のガロア群** の元。具体的な対応は:

- $k = 5$ のべき: **巡回回転**（$\mathbb{Z}_t[X]/(X^n+1)$ で $k = 5^j$ の作用）
- $k = -1$（i.e. $X \to X^{-1} = X^{2n-1}$): **共役**（$\mathbb{Z}_t^*$ の逆順）

これらを組み合わせて所望の置換を実現。

### 5.4 Rotation key

$X \to X^k$ の作用を暗号文に適用するには、**秘密鍵の変換** が必要:

$$
s(X) \to s(X^k) =: s_k
$$

暗号文 $(c_0, c_1)$ の $X \to X^k$ 変換は $(c_0(X^k), c_1(X^k))$ で、これを $s_k$ で復号すると所望の置換平文になる。

しかし $s_k$ と $s$ は異なる鍵。**鍵スイッチで $s_k \to s$** に戻す:

```
rot_key_k = Enc_s(s_k)   # 各 k ごとに用意
```

### 5.5 コスト

- **1 rotation ≈ 1 key switch** のコスト
- メモリ: 各 $k$ に対して rot_key が必要（数 MB/key）

$n$ 回転すべてを用意すると膨大。実用では **pow of 2 rotation** のみ用意し、他は組み合わせで対応。

## 6. ガロア自動化と key switch

### 6.1 Key switch の抽象化

Article 8 で再線形化を「鍵スイッチ」と見たが、実は **任意の変換** に一般化できる:

- $s^2 \to s$: 再線形化
- $s(X^k) \to s(X)$: 回転
- 異なるパラメータ $s_{\text{new}} \to s_{\text{old}}$: bootstrap で使う

これらすべてを **Enc_s(target secret)** をビット分解して対応。

### 6.2 Key switch のコスト

```
Cost ≈ O(n log n · L)   # L は decomposition level
```

乗算や回転のたびに key switch が走る → **FHE 計算の主要コスト**。NTTは高速だが、key switch は定数倍遅い。

### 6.3 ハイブリッド key switch

SEAL や OpenFHE は **hybrid key switching** を実装:

1. **BV (Brakerski-Vaikuntanathan) key switch**: gadget decomposition 型、シンプル
2. **GHS key switch**: モジュラス拡張型、高速だがメモリ多い

ハイブリッドは両者を組み合わせて最適化。

## 7. パラメータ選択

### 7.1 HE Standard の推奨

| $\log_2 q$ | $n$ | 安全性 | 乗算深さ (目安) |
|---|---|---|---|
| 54 | 2048 | 128 | 1 |
| 109 | 4096 | 128 | 3 |
| 218 | 8192 | 128 | 10 |
| 438 | 16384 | 128 | 25 |
| 881 | 32768 | 128 | 60 |

### 7.2 $t$ の選び方

- **$t$ 小さく**: 平文空間が狭い、計算失敗が少ない
- **$t$ 大きく**: 多項式近似などで精度 UP、ただしノイズ大
- **$t \equiv 1 \pmod{2n}$**: SIMD バッチングに必須
- 典型的: $t = 65537, 786433, 1179649$

### 7.3 BFV vs BGV でのパラメータ選び

基本は同じ。違いは:

- BFV: $\Delta = q/t$ が自然に決まる
- BGV: モジュラスチェーンの刻み幅 $p_i$ を選ぶ

実装ライブラリが自動で適切な値を提案する。

## 8. Microsoft SEAL API 例

### 8.1 BFV での基本操作

```cpp
#include "seal/seal.h"
using namespace seal;

EncryptionParameters parms(scheme_type::bfv);
parms.set_poly_modulus_degree(8192);
parms.set_coeff_modulus(CoeffModulus::BFVDefault(8192));
parms.set_plain_modulus(1024);

SEALContext context(parms);
KeyGenerator keygen(context);
SecretKey secret_key = keygen.secret_key();
PublicKey public_key; keygen.create_public_key(public_key);
RelinKeys relin_keys; keygen.create_relin_keys(relin_keys);
GaloisKeys galois_keys; keygen.create_galois_keys(galois_keys);

Encryptor encryptor(context, public_key);
Evaluator evaluator(context);
Decryptor decryptor(context, secret_key);
BatchEncoder encoder(context);

// バッチエンコード
vector<uint64_t> input(encoder.slot_count(), 0);
for (size_t i = 0; i < 10; i++) input[i] = i + 1;

Plaintext plain; encoder.encode(input, plain);
Ciphertext ct; encryptor.encrypt(plain, ct);

// 平方化
Ciphertext ct_squared;
evaluator.square(ct, ct_squared);
evaluator.relinearize_inplace(ct_squared, relin_keys);

// 復号
Plaintext plain_out; decryptor.decrypt(ct_squared, plain_out);
vector<uint64_t> output; encoder.decode(plain_out, output);

// output[0..9] = 1, 4, 9, 16, 25, ..., 100
```

### 8.2 回転の例

```cpp
// スロット回転
Ciphertext ct_rotated;
evaluator.rotate_rows(ct, 1, galois_keys, ct_rotated);
// 1 シフト: [a_1, a_2, ..., a_n] → [a_2, a_3, ..., a_n, a_1]
```

### 8.3 全スロット和

```cpp
Ciphertext ct_sum = ct;
for (size_t shift = 1; shift < encoder.slot_count() / 2; shift *= 2) {
    Ciphertext temp;
    evaluator.rotate_rows(ct_sum, shift, galois_keys, temp);
    evaluator.add_inplace(ct_sum, temp);
}
// ct_sum の各スロット = 元の全スロットの和
```

## 9. Q&A

### Q1: BGV と BFV、どっちを使えばいい？

**どちらでもよい**。SEAL では BFV がデフォルト。BGV は **$t$ を大きくしたい場合** に若干有利。

### Q2: SIMD バッチング は自動？

**自動ではない**。`BatchEncoder` で明示的にエンコード/デコード。

### Q3: なぜ $t \equiv 1 \pmod{2n}$ が必要？

SIMD バッチングのために $X^n + 1$ が $\bmod t$ で **$n$ 個の 1 次因子に分解** される必要がある。これには $t \equiv 1 \pmod{2n}$ が必要。

### Q4: Rotation key のメモリが大きすぎる

**対策**:
- Pow of 2 のみ生成 → 他は組み合わせ（遅いが軽い）
- 必要な回転だけ生成
- ハイブリッド key switch で圧縮

### Q5: BGV の Bootstrapping は実装されている？

**SEAL では未実装**（2024 年時点）。**OpenFHE では実装**。BGV Bootstrap は数秒オーダーで、通常は Leveled で運用する。

### Q6: BFV の乗算でノイズが想像以上に増える

**通常より大きい noise 増加**を招く原因:
- $t$ が大きすぎる
- 乗算後に **再線形化を忘れている** (3 元暗号文のまま)
- 連続乗算で noise budget を使い切った

SEAL の `parms_id` や `modulus_switch_to_next` で Level 管理を正しく。

## 10. まとめ

### 本記事で学んだこと

- **BGV** と **BFV** は整数演算 FHE の 2 大スキーム。Ring-LWE ベース
- **BGV**: 平文が下位側、ノイズに $t$ が掛かる。Mod switch で管理
- **BFV**: 平文が上位側（$\Delta m$）、スケーリングで管理
- 実用上ほぼ同等。SEAL では BFV がデフォルト
- **SIMD バッチング** (CRT packing): 1 暗号文に $n$ 個の独立平文
- **Rotation**: ガロア自動化 $X \to X^k$ + 鍵スイッチ
- **パラメータ**: $n = 2048 \sim 16384$、$t \equiv 1 \pmod{2n}$

### 次の記事（Article 12）へ

次の記事では **CKKS** を扱う。BGV/BFV が整数演算なのに対し、CKKS は **実数/複素数の近似演算**。ML 推論に最適化され、現代 PPML で最も使われるスキーム。エンコーディングの違い（canonical embedding）、rescale の意味、precision vs noise のトレードオフを詳しく見る。

### 3行サマリ

- **BGV/BFV** は整数演算 FHE。Ring-LWE ベースで、SIMD バッチングと rotation が武器
- 平文位置の違い（下位 vs 上位）と mod switch の扱いだけが本質的な違い
- **1 つの暗号文で $n$ 個の並列計算** ができる → ベクトル・行列計算に絶大な威力

---

## 参考文献

- Brakerski, Gentry, Vaikuntanathan. *(Leveled) Fully Homomorphic Encryption without Bootstrapping.* ITCS 2012.
- Fan, Vercauteren. *Somewhat Practical Fully Homomorphic Encryption.* IACR ePrint 2012.
- Smart, Vercauteren. *Fully Homomorphic SIMD Operations.* Designs, Codes and Cryptography, 2014.
- Microsoft SEAL. [https://github.com/microsoft/SEAL](https://github.com/microsoft/SEAL)
- OpenFHE. [https://github.com/openfheorg/openfhe-development](https://github.com/openfheorg/openfhe-development)
