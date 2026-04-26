---
title: "[完全準同型暗号入門シリーズ 13/15] TFHE — 高速Bootstrapと任意関数評価"
emoji: "⚡"
type: "tech"
topics:
  - fhe
  - tfhe
  - bootstrap
  - pbs
  - boolean
published: false
---

**日付**: 2026年4月24日
**学習内容**: **TFHE (Torus Fully Homomorphic Encryption)** は、Chillotti-Gama-Georgieva-Izabachène (CGGI, 2016-2020) が提案した **ゲートベース** の FHE スキーム。BGV/BFV/CKKS が「バッチ単位で計算、最後に bootstrap」なのに対し、TFHE は「**1 ゲートごとに bootstrap**」を **13 ms** の高速で実行する。さらに **Programmable Bootstrapping (PBS)** により、**任意の 1 変数関数** を bootstrap と同じコストで評価できる。本記事では **(1) TFHE の torus ベース表現**、**(2) LWE 暗号文と RGSW 暗号文**、**(3) Blind Rotation の原理**、**(4) Gate Bootstrapping**、**(5) Programmable Bootstrapping**、**(6) TFHE-rs と Concrete でのコード例**、**(7) TFHE vs CKKS の使い分け** を扱う。

## 0. 本記事の位置づけ

BGV/BFV/CKKS は「バッチ計算」に特化する一方で、**個別の bit 演算や非線形判定** が苦手だった。TFHE はその逆で:

- **1 bit ごと**（または小さい整数ごと）の計算が得意
- **任意の 1 変数関数** を PBS で評価できる
- ただし SIMD バッチングは弱い

この特性から:

- **Boolean 回路**（暗号化 CPU のような）: TFHE が最適
- **深い NN 推論のベクトル化**: CKKS が最適
- **ハイブリッド**（例: CKKS で行列演算、TFHE で活性化関数）: 最近のトレンド

TFHE は Zama 社が **TFHE-rs** (Rust) と **Concrete** (Python 向けコンパイラ) を OSS で提供。fhEVM（FHE Ethereum）の中核技術。

構成:

- **第1章**: TFHE の基本哲学
- **第2章**: Torus $\mathbb{T}$ 上の暗号
- **第3章**: LWE vs GLWE vs RGSW
- **第4章**: Blind Rotation
- **第5章**: Gate Bootstrapping
- **第6章**: Programmable Bootstrapping
- **第7章**: TFHE-rs コード例
- **第8章**: TFHE vs CKKS
- **第9章**: Q&A とまとめ

## 1. TFHE の基本哲学

### 1.1 考え方の逆転

BGV/BFV/CKKS: **計算可能な深さ $L$ を最大化 → bootstrap を遅らせる**
TFHE: **各 gate ごとに即 bootstrap → 乗算深さの概念がない**

TFHE の発想:

> **「Bootstrap が 13 ms なら、毎ゲート bootstrap すれば良い」**

これにより:
- **任意の深さの回路** が実行可能
- **ノイズ管理が自動化**（毎回リセットされる）
- **Leveled と FHE の区別がない**

### 1.2 平文空間

TFHE の平文は **1 bit** または **小さい整数** ($\mathbb{Z}_p$ for small $p$):

- Boolean: $\{0, 1\}$
- Integer: $\{0, 1, \ldots, p-1\}$ for $p = 4, 8, 16$
- Torus: $\mathbb{T} = \mathbb{R}/\mathbb{Z}$ (上を連続的にみたもの)

BFV/CKKS の「ベクトル平文」ではなく **スカラー平文**が基本。

### 1.3 Boolean gate 支援

TFHE は Boolean 論理ゲートをネイティブに提供:

- `nand(ct_1, ct_2)` → Enc(NAND(m_1, m_2))
- `and, or, xor, not, mux`

これで **暗号化された CPU** のような任意計算が可能。

## 2. Torus $\mathbb{T}$ 上の暗号

### 2.1 Torus とは

$\mathbb{T} = \mathbb{R}/\mathbb{Z}$、つまり 0 と 1 を同一視した **「円環状の実数」**。

実装上は $\mathbb{Z}_q$ を使うが、概念的には torus で考える。$\mathbb{Z}_q$ の要素 $a$ は $a/q \in \mathbb{T}$ に対応。

### 2.2 なぜ torus か

- **加法が自然**（mod 1 で閉じる）
- **乗算も部分的に自然**（整数 × torus は OK、torus × torus は未定義）
- **量子化が扱いやすい**（$\mathbb{T}$ を $p$ 分割することで $\mathbb{Z}_p$ 平文を作る）

### 2.3 LWE 暗号文 (over torus)

$\mathbb{T}$ 上の LWE 暗号文:

$$
\text{ct} = (\mathbf{a}, b) \in \mathbb{T}^n \times \mathbb{T}
$$

$$
b = \langle \mathbf{a}, \mathbf{s}\rangle + e + m/p
$$

ここで $\mathbf{s} \in \{0, 1\}^n$ (binary)、$m \in \mathbb{Z}_p$、$e$ は小ノイズ。

復号:

$$
b - \langle \mathbf{a}, \mathbf{s}\rangle = m/p + e \approx m/p
$$

最も近い $\mathbb{Z}_p$ の元を取れば $m$ が得られる。

## 3. LWE vs GLWE vs RGSW

TFHE では **3 種類** の暗号文を使い分ける。

### 3.1 LWE (スカラー暗号)

上記の形。**スカラー平文** に対応。1 bit または小さい整数。

### 3.2 GLWE (一般化 LWE)

Ring-LWE を拡張:

$$
\text{ct} = (\mathbf{A}, B) \in (R_\mathbb{T})^k \times R_\mathbb{T}
$$

$R_\mathbb{T} = \mathbb{T}[X]/(X^N+1)$、$\mathbf{s} \in R^k$。

- $k = 1$, $N > 1$: Ring-LWE (RLWE)
- $k > 1$, $N = 1$: LWE (with $k$-dim secret)
- $k = 1$, $N = 1$: 退化ケース（実用しない）

平文は **多項式** $m \in R_\mathbb{T}$。

### 3.3 RGSW (Ring GSW)

Gentry-Sahai-Waters (GSW) 暗号の ring 版。**行列暗号文**:

$$
\text{RGSW}(m) = \text{Enc}(m) \cdot G
$$

ここで $G$ は **gadget matrix**。RGSW は **GLWE 暗号文との乗算** が「ノイズ増加が加算的」という優れた性質を持つ。これが blind rotation の核。

## 4. Blind Rotation

### 4.1 目的

LWE 暗号文 $(\mathbf{a}, b) = \text{Enc}(m/p + e)$ が与えられたとき、**$m$ を見ずに** 多項式 $X^{b - \langle \mathbf{a}, \mathbf{s}\rangle}$ を計算したい。

結果の多項式の **定数項** が所望の出力（lookup table を設計すれば任意 $f(m)$）。

### 4.2 アルゴリズム

1. **テストベクトル** $v(X)$ を用意（後述）
2. **初期化**: $\text{ACC} = X^{\tilde{b}} \cdot v(X)$（$\tilde{b}$ は $b$ の量子化）
3. **各 $i$ で**: $\text{ACC} = \text{CMux}(\text{RGSW}(s_i), X^{-\tilde{a}_i} \cdot \text{ACC}, \text{ACC})$
4. 最終 ACC の **定数項** を取り出す

**Clever なのは 3 の CMux**:
- $s_i = 1$ → $\text{ACC} \leftarrow X^{-\tilde{a}_i} \cdot \text{ACC}$
- $s_i = 0$ → $\text{ACC}$ そのまま

結果として $\text{ACC}$ は $X^{\tilde{b} - \sum \tilde{a}_i s_i} \cdot v(X) = X^{\tilde{m/p + e}} \cdot v(X)$ となる。

### 4.3 テストベクトル $v(X)$

$v(X)$ を適切に設計すると、**任意の関数 $f$** を実現できる。たとえば:

$$
v(X) = \sum_{j=0}^{N-1} f(j) \cdot X^j
$$

ACC が $X^k v(X)$ になったとき、定数項は **$f(-k \bmod N)$**。

つまり **$f$ の値を埋め込んだルックアップテーブル** として $v$ が機能。

### 4.4 Bootstrapping Key

RGSW の形で **秘密鍵の各 bit** $s_i$ を暗号化したもの:

$$
\text{bsk} = (\text{RGSW}(s_1), \text{RGSW}(s_2), \ldots, \text{RGSW}(s_n))
$$

これが bootstrapping key。サイズは数十 MB。

## 5. Gate Bootstrapping

### 5.1 Bootstrap の意味

LWE 暗号文 $(a, b)$ で $m \in \{0, 1\}$ を暗号化しているとする。

Gate Bootstrapping:

1. Blind rotation で $v(X) = \sum f(j) X^j$ を設定
2. $f$ を **論理ゲート関数** に選ぶ（例: $f(x) = x \bmod 2$ で identity）
3. 結果: $\text{Enc}(f(m))$ で、**ノイズは新鮮**

### 5.2 2入力ゲートの実装

NAND ゲートを暗号化したまま計算する例:

1. 2 つの入力 $\text{ct}_1, \text{ct}_2 \in \text{LWE}(\{0,1\})$
2. **線形結合**: $\text{ct}_{\text{lin}} = \text{ct}_1 + \text{ct}_2$（平文 $\in \{0, 1, 2\}$）
3. **PBS**: $f(x) = \text{NAND}(x == 0, x == 1)$ を lookup で実装

これで **13 ms** で NAND ゲート完了。

### 5.3 他のゲート

- AND: $f(x) = (x == 2)$
- OR: $f(x) = (x \geq 1)$
- XOR: 別の符号化で
- NOT: 平文 $-1$ (torus 上で反転)

### 5.4 TFHE の速度比較

| スキーム | 1 gate | 1 乗算 (SIMD $n=4096$) |
|---|---|---|
| BGV | — | 数 ms (バッチ) |
| BFV | — | 数 ms (バッチ) |
| CKKS | — | 数 ms (バッチ) |
| **TFHE** | **13 ms** | — (SIMD 弱い) |

**粒度の違い** が本質。

## 6. Programmable Bootstrapping

### 6.1 PBS = 任意関数評価

Blind rotation の **テストベクトル $v(X)$** を任意に選べる → **任意の 1 変数関数** が bootstrap コストで評価可能:

$$
\text{PBS}_f(\text{ct}) = \text{Enc}(f(m))
$$

### 6.2 応用例

- **ReLU**: $f(x) = \max(0, x)$ を lookup
- **Sigmoid**: $f(x) = \sigma(x)$ を離散化して lookup
- **Comparison**: $f(x) = [x > 0]$
- **Modulo**: $f(x) = x \bmod k$

**多項式近似が不要** → 精度 UP + 速度 UP。

### 6.3 制約

- 入力が **小さい整数**（$\mathbb{Z}_p$, $p \leq 32$ 程度）
- 関数も **離散** (lookup table のサイズ $= N \geq 2^{10}$)
- 多変数関数は **工夫が必要**（2D lookup を 2 ステップで）

### 6.4 最近の進展

- **2-input PBS** (2023): 2 変数関数を 1 PBS で評価
- **Multi-bit TFHE**: 平文を 4-8 bit に拡張
- **CKKS-TFHE hybrid**: CKKS で線形層、TFHE で活性化

## 7. TFHE-rs コード例

### 7.1 TFHE-rs (Rust)

Zama 社が開発する TFHE の Rust 実装。

```rust
use tfhe::prelude::*;
use tfhe::{generate_keys, set_server_key, ConfigBuilder, FheUint8};

fn main() {
    // パラメータとキー
    let config = ConfigBuilder::default().build();
    let (client_key, server_key) = generate_keys(config);

    // サーバーが演算に使うキーを設定
    set_server_key(server_key);

    // 暗号化
    let a = FheUint8::encrypt(42u8, &client_key);
    let b = FheUint8::encrypt(100u8, &client_key);

    // 暗号化されたまま加算・乗算・比較
    let sum = &a + &b;           // Enc(142)
    let prod = &a * &b;           // Enc(42 * 100 mod 256)
    let gt = a.gt(&b);            // Enc(42 > 100 = false = 0)

    // 復号
    let sum_plain: u8 = sum.decrypt(&client_key);
    let prod_plain: u8 = prod.decrypt(&client_key);
    let gt_plain: bool = gt.decrypt(&client_key);

    println!("{} + {} = {}", 42, 100, sum_plain);  // 142
    println!("{} * {} = {}", 42, 100, prod_plain); // 200 (mod 256 なので!)
    println!("{} > {} = {}", 42, 100, gt_plain);   // false
}
```

### 7.2 Concrete (Python)

Zama 社のもう一つの提供物。**Python コードを FHE 回路にコンパイル**:

```python
from concrete import fhe

@fhe.compiler({"x": "encrypted", "y": "encrypted"})
def f(x, y):
    return (x + y) * 2 - 1

# 訓練データでコンパイル
inputset = [(i, j) for i in range(0, 16) for j in range(0, 16)]
circuit = f.compile(inputset)

# 暗号化 → 実行 → 復号
x_enc = circuit.encrypt(3)
y_enc = circuit.encrypt(5)
result_enc = circuit.run(x_enc, y_enc)
result = circuit.decrypt(result_enc)
# (3 + 5) * 2 - 1 = 15
```

Concrete は内部で:
1. Python AST を TFHE 回路に変換
2. LUT を自動生成して PBS に展開
3. 最適化された TFHE-rs 呼び出し

### 7.3 応用例

- **暗号化 CPU エミュレーション**: RISC-V の命令を TFHE で実装
- **私的ML**: 暗号化 NN 推論（線形層 + PBS 活性化）
- **ブロックチェーン**: Zama fhEVM で Solidity スマコンを暗号化したまま実行

## 8. TFHE vs CKKS

### 8.1 得意分野

| 項目 | CKKS | TFHE |
|---|---|---|
| 行列ベクトル積 | **速い** (SIMD) | 遅い |
| 非線形関数 | 多項式近似 (重い) | **PBS で直接** |
| Boolean 回路 | 非効率 | **ネイティブ** |
| 精度 | 30-40 bit | 完全 (低 bit 平文) |
| 深い計算 | 要 bootstrap | **自動** |

### 8.2 ハイブリッドの登場

最近 (2023-2025) のトレンド:

**NN 推論のハイブリッド**:
- 線形層 (行列積) → CKKS
- 活性化関数 (ReLU) → TFHE PBS
- 間で scheme 変換 (**scheme switching**)

これで「両方の良いとこ取り」が実現しつつある。**CHIMERA**, **Pegasus** などが有名な研究。

### 8.3 どちらを学ぶべき？

- **ML に集中** → CKKS
- **Boolean/整数 計算** → TFHE
- **汎用** → 両方、将来ハイブリッド

現代のプラクティスでは TFHE の知識が **増加傾向**（Zama が強力に推進中、fhEVM の需要）。

## 9. Q&A

### Q1: TFHE は遅い？速い？

**ゲート単位では速い (13ms)**、**バッチ単位では遅い**。処理データの性質で判断。

### Q2: TFHE で ML 推論はできる？

**可能だが遅い**。1 bit/1 integer ずつの処理が基本なので、深い NN は現実的でない。**PBS で活性化関数** を評価するのは強力。

### Q3: Multi-bit TFHE とは？

平文を **4 ビット、8 ビット** など拡大した TFHE 変種。整数演算が効率化されるが、PBS コストが増える。

### Q4: fhEVM とは？

Zama の **EVM 互換 FHE 仮想マシン**。Solidity で `euint32` (encrypted uint32) のような型を使い、暗号化されたまま演算する。

### Q5: TFHE の秘密鍵は binary？

**通常は binary** ({0, 1}) または **ternary** ({-1, 0, 1})。バイナリーは解析が楽、ternary はノイズ減。

### Q6: PBS のコストを減らす方法は？

- **大きな $N$** → 1 PBS で大きな LUT
- **Multi-output PBS** → 1 PBS で複数の関数
- **Circuit bootstrapping** → PBS を回路化

いずれも活発な研究領域。

## 10. まとめ

### 本記事で学んだこと

- **TFHE** = ゲートベース FHE。**1 gate ≈ 13 ms で bootstrap**
- Torus $\mathbb{T}$ 上で定義され、**LWE / GLWE / RGSW** の 3 暗号文を使い分け
- **Blind Rotation + gadget decomposition** が bootstrap の核
- **Programmable Bootstrapping (PBS)**: 任意の 1 変数関数を bootstrap と同コストで評価
- Boolean 回路、整数演算、非線形関数に強い
- **TFHE-rs (Rust)、Concrete (Python)** が実装ライブラリ
- CKKS とのハイブリッドが次世代のトレンド

### 次の記事（Article 14）へ

次の記事では **FHE ライブラリと実装** を俯瞰する。Microsoft SEAL、OpenFHE、HElib、TFHE-rs、Concrete、Lattigo を比較し、実際に手を動かすコード例（Python + OpenFHE）で加算・乗算・rotation・bootstrap を体験する。

### 3行サマリ

- **TFHE** = 1 gate を 13 ms で bootstrap する爆速スキーム
- **PBS (Programmable Bootstrapping)** で任意 1 変数関数が bootstrap コストで評価可能
- Boolean/整数/非線形が得意 → **Zama fhEVM、暗号化 CPU、FHE-ML 活性化層** の主役

---

## 参考文献

- Chillotti, Gama, Georgieva, Izabachène. *TFHE: Fast Fully Homomorphic Encryption over the Torus.* Journal of Cryptology, 2020.
- Chillotti et al. *Programmable Bootstrapping Enables Efficient Homomorphic Inference of Deep Neural Networks.* CSCML 2021.
- Zama. *TFHE-rs Documentation.* [https://docs.zama.ai/tfhe-rs](https://docs.zama.ai/tfhe-rs)
- Zama. *Concrete Documentation.* [https://docs.zama.ai/concrete](https://docs.zama.ai/concrete)
- Boura, Gama, Georgieva, Jetchev. *CHIMERA: Combining Ring-LWE-based Fully Homomorphic Encryption Schemes.* J. Math. Crypt. 2020.
