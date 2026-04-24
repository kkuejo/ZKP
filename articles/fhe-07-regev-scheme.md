---
title: "[完全準同型暗号入門シリーズ 7/15] Regev暗号方式 — LWEから作る最初の暗号"
emoji: "🔑"
type: "tech"
topics:
  - fhe
  - regev
  - lwe
  - publickey
  - encryption
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事では Regev 2005 の**公開鍵暗号方式**を、アルゴリズム・数式・簡易的な Python 風擬似コードを交えて最後まで追う。具体的には **(1) パラメータ選択**、**(2) 鍵生成 (KeyGen)**、**(3) 暗号化 (Enc)**、**(4) 復号 (Dec)**、**(5) 正当性の証明**、**(6) IND-CPA 安全性のスケッチ**、**(7) 準同型性の芽生え**、そして **(8) Ring-LWE 版 Regev** への自然な拡張。ここまで来て初めて「暗号文 = 格子点 + ノイズ」「復号 = 内積で格子点を除去してノイズだけ残す」という FHE のコアメカニズムが明確になる。Article 8 以降で扱う **加算・乗算の準同型性** と **ノイズ管理** の土台となる。

## 0. 本記事の位置づけ

Article 5-6 で LWE と Ring-LWE を学んだ。本記事ではそれらを実際の暗号スキームに落とし込む。Regev 2005 の原論文は「LWE 仮定から公開鍵暗号を構築できる」ことを示した**歴史的論文**。

この暗号は:

- **1 ビット** の平文を暗号化
- **加法的に準同型** → 暗号文を足すと平文の和の暗号文になる（ただしノイズが増える）
- **IND-CPA 安全**

後の BGV/BFV/CKKS は、このスキームを起点に「**複数ビット・多項式平文・深い乗算・ブートストラッピング**」を次々と積み上げていく。いわば「FHE の出発駅」。

構成:

- **第1章**: パラメータと記号
- **第2章**: 鍵生成
- **第3章**: 暗号化
- **第4章**: 復号と正当性
- **第5章**: IND-CPA 安全性
- **第6章**: 準同型加算
- **第7章**: Python 擬似コード
- **第8章**: Ring-LWE 版への発展
- **第9章**: Q&A とまとめ

## 1. パラメータと記号

### 1.1 パラメータ一覧

| 記号 | 意味 | 典型値 |
|---|---|---|
| $n$ | LWE の次元 | 1024 |
| $q$ | モジュラス（素数でもよい） | $\approx 2^{32}$ |
| $m$ | 公開鍵のサンプル数 | $\approx n \log q$ |
| $\sigma$ | 誤差分布の標準偏差 | $3.19$ |
| $\chi$ | 離散ガウス分布 $D_{\mathbb{Z},\sigma}$ | |
| $\mathbf{s}$ | 秘密鍵 | $\mathbb{Z}_q^n$ |

### 1.2 平文空間

Regev オリジナルは **1 ビット** 平文:

$$
m \in \{0, 1\}
$$

復号時に「0 or 1」を識別できれば OK。

### 1.3 メンタルモデル

> **「公開鍵 = ノイズ付き線形方程式の束」**
> **「暗号文 = それらのランダム部分和 + 平文情報」**
> **「復号 = 秘密鍵で内積を取って、ノイズと平文を分離」**

これを頭に置いて以下を読む。

## 2. 鍵生成 (KeyGen)

### 2.1 アルゴリズム

```
KeyGen(n, q, m, σ):
    # 秘密鍵
    s ← Uniform(ℤ_q^n)
    
    # 公開鍵
    A ← Uniform(ℤ_q^{m × n})          # m × n ランダム行列
    e ← χ^m                            # m 次元ノイズベクトル
    b ← A · s + e  (mod q)            # LWE サンプル
    
    pk = (A, b)
    sk = s
    return (pk, sk)
```

### 2.2 数学的に書くと

秘密鍵 $\mathbf{s} \in \mathbb{Z}_q^n$ はランダム。

公開鍵:

$$
A \in \mathbb{Z}_q^{m \times n}, \quad \mathbf{e} \in \mathbb{Z}^m \sim \chi^m, \quad \mathbf{b} = A\mathbf{s} + \mathbf{e} \pmod q
$$

$(A, \mathbf{b})$ の組は、LWE のサンプルの集合そのもの。

### 2.3 なぜ $m$ が大きい？

$m$ が小さいと:
- 攻撃者が LWE を破るのに情報が足りない → **でも暗号化側も情報が足りない**

$m \approx n \log q$ 程度にすることで:
- 暗号化時に十分な部分集合和が取れる
- LWE の困難性は $m$ に依存せず保たれる（$m = \text{poly}(n)$ で定数時間攻撃がない）

## 3. 暗号化 (Enc)

### 3.1 アルゴリズム

```
Enc(pk, μ ∈ {0,1}):
    # ランダムな部分集合（実装では 0/1 ベクトル）
    r ← Uniform({0,1}^m)
    
    # サンプルの部分和
    c_1 ← Aᵀ · r  (mod q)         # ∈ ℤ_q^n
    c_2 ← <b, r> + μ · ⌊q/2⌋  (mod q)   # ∈ ℤ_q
    
    return (c_1, c_2)
```

### 3.2 数学的に書くと

ランダムな 0/1 ベクトル $\mathbf{r} \in \{0,1\}^m$ を選ぶ。

$$
\mathbf{c}_1 = A^T \mathbf{r} = \sum_{i: r_i = 1} \mathbf{a}_i \in \mathbb{Z}_q^n
$$

$$
c_2 = \langle \mathbf{b}, \mathbf{r} \rangle + \mu \cdot \lfloor q/2 \rfloor = \sum_{i: r_i = 1} b_i + \mu \cdot \lfloor q/2 \rfloor \pmod q
$$

暗号文は $(\mathbf{c}_1, c_2) \in \mathbb{Z}_q^n \times \mathbb{Z}_q$。

### 3.3 直感的には

- $\mathbf{c}_1$: **乱択したサンプルの $\mathbf{a}_i$ の和**
- $c_2$: **対応する $b_i$ の和 + 平文シフト $\mu \cdot q/2$**

平文が 0 なら $c_2 = \sum b_i$、平文が 1 なら $c_2 = \sum b_i + q/2$。

## 4. 復号と正当性

### 4.1 アルゴリズム

```
Dec(sk, (c_1, c_2)):
    v ← c_2 - <c_1, s>  (mod q)
    
    if v is close to 0:
        return 0
    else if v is close to q/2:
        return 1
```

「$v$ が 0 に近い」は、$|v| < q/4$ で判定（$v \in (-q/2, q/2]$ の表現で）。

### 4.2 正当性の証明

$v = c_2 - \langle \mathbf{c}_1, \mathbf{s}\rangle$ を展開:

$$
\begin{aligned}
v &= \langle \mathbf{b}, \mathbf{r}\rangle + \mu \lfloor q/2 \rfloor - \langle A^T \mathbf{r}, \mathbf{s}\rangle \\
&= \langle \mathbf{b}, \mathbf{r}\rangle + \mu \lfloor q/2 \rfloor - \langle \mathbf{r}, A\mathbf{s}\rangle \\
&= \langle \mathbf{b} - A\mathbf{s}, \mathbf{r}\rangle + \mu \lfloor q/2 \rfloor \\
&= \langle \mathbf{e}, \mathbf{r}\rangle + \mu \lfloor q/2 \rfloor
\end{aligned}
$$

ここで $\mathbf{e} \sim \chi^m$ は小さいベクトル、$\mathbf{r} \in \{0,1\}^m$ は 0/1 ベクトル。したがって:

$$
\langle \mathbf{e}, \mathbf{r} \rangle = \sum_{i: r_i = 1} e_i
$$

は $|\mathbf{r}|$ 個（平均 $m/2$ 個）の小さなガウス値の和 → **$O(\sqrt{m/2} \cdot \sigma)$ のオーダー**。

復号が正しく動くには:

$$
|\langle \mathbf{e}, \mathbf{r}\rangle| < q/4
$$

が必要。つまり:

$$
\sqrt{m} \cdot \sigma < q/4
$$

これを満たすよう $q, \sigma, m$ を選ぶ。

### 4.3 失敗確率

ガウス分布の裾を考えると、$|v - \mu q/2| > q/4$ となる確率は極めて小さい（$2^{-40}$ などのオーダー）。実用上は「**ほぼ確実に正しく復号される**」。

### 4.4 図解

```
0              q/4              q/2              3q/4              q
|───────────────|────────────────|────────────────|────────────────|
      μ=0                     μ=1
    v = 0 付近              v = q/2 付近
    
ノイズが q/4 以内なら μ を正しく判定できる
```

## 5. IND-CPA 安全性

### 5.1 安全性定義

**IND-CPA (Indistinguishability under Chosen Plaintext Attack)**:

> 攻撃者が平文 $m_0, m_1$ を選び、$\text{Enc}(m_b)$ を受け取る（$b$ はランダム）。攻撃者は $b$ を当てられるか？

「当てる確率 = 1/2 + 無視可能」なら IND-CPA 安全。

### 5.2 Regev スキームの安全性

**証明のスケッチ**:

1. **Hybrid 1**: 本物の公開鍵 $\mathbf{b} = A\mathbf{s} + \mathbf{e}$ を、**一様ランダム** $\mathbf{b}'$ に置き換える
   - Decision-LWE 仮定で区別不可能
2. **Hybrid 2**: 暗号文 $(\mathbf{c}_1, c_2)$ も、$\mathbf{b}$ が一様なので **一様ランダム**
   - 平文 $\mu$ の情報は完全に消える
3. したがって **攻撃者は $\mu$ を判別できない**

### 5.3 重要な仮定

- **Decision-LWE 仮定**: 本質的な仮定
- **$r$ の分布**: 0/1 一様でよい（整数係数でもよい）
- **$\mathbf{e}$ の分布**: ガウス分布（他の分布でも OK だが解析が楽）

### 5.4 チャレンジ攻撃には対応しない

IND-CPA は「受動的攻撃」のみに対する安全性。**IND-CCA (能動的攻撃)** では安全でない:

- 攻撃者が暗号文をわずかに変形して復号オラクルに問い合わせ → 平文を推測

FHE 全般は **CCA 安全ではない**（準同型性と CCA 安全性は本質的に両立しない）。実用では **別の手段（署名、ゼロ知識証明）** で CCA 的性質を補う。

## 6. 準同型加算

### 6.1 加算の準同型性

2 つの暗号文 $(c_1, c_2)$ と $(c_1', c_2')$ について、成分ごとに足すと:

$$
(c_1 + c_1',\ c_2 + c_2')
$$

この暗号文を復号すると:

$$
v_{\text{sum}} = (c_2 + c_2') - \langle c_1 + c_1', s\rangle = (\langle e, r \rangle + \mu q/2) + (\langle e, r' \rangle + \mu' q/2) = \langle e, r+r'\rangle + (\mu + \mu') q/2
$$

**$\mu + \mu'$ の暗号文になる**（ノイズも足し合わさっている）。

### 6.2 ノイズの増加

加算するとノイズは $\sqrt{2}$ 倍くらいに増える。$L$ 回加算で $\sqrt{L}$ 倍に。

復号失敗を避けるには:

$$
\sqrt{L} \cdot \sqrt{m} \cdot \sigma < q/4
$$

$L$ 回の加算をサポートするには $q$ を大きくする必要。

### 6.3 乗算はどうする？

Regev スキームは **加算のみ** 準同型。乗算は素直にはできない。

乗算を可能にするには:
1. **暗号文のテンソル化**: $(c_1, c_2) \otimes (c_1', c_2')$ → サイズが膨張
2. **再線形化 (relinearization)**: テンソル化された暗号文を元のサイズに戻す

これが BGV/BFV/CKKS の核心で、Article 8 以降で扱う。

## 7. Python 擬似コード

### 7.1 完全実装例

```python
import numpy as np

# パラメータ
n = 512       # LWE 次元
q = 2**20     # モジュラス
m = n * 2     # サンプル数
sigma = 3.19  # ノイズ

def discrete_gaussian(size, sigma):
    """離散ガウス分布からのサンプリング（近似）"""
    return np.round(np.random.normal(0, sigma, size)).astype(np.int64)

def keygen():
    s = np.random.randint(0, q, size=n)
    A = np.random.randint(0, q, size=(m, n))
    e = discrete_gaussian(m, sigma)
    b = (A @ s + e) % q
    return (A, b), s

def encrypt(pk, mu):
    A, b = pk
    r = np.random.randint(0, 2, size=m)          # 0/1 ベクトル
    c1 = (A.T @ r) % q                           # ∈ ℤ_q^n
    c2 = (b @ r + mu * (q // 2)) % q             # ∈ ℤ_q
    return c1, c2

def decrypt(sk, ct):
    c1, c2 = ct
    v = (c2 - c1 @ sk) % q
    # v を (-q/2, q/2] の符号付きに変換
    if v > q // 2:
        v -= q
    # 閾値判定
    return 0 if abs(v) < q // 4 else 1

# デモ
pk, sk = keygen()
c0 = encrypt(pk, 0)
c1 = encrypt(pk, 1)
print(decrypt(sk, c0))  # → 0
print(decrypt(sk, c1))  # → 1

# 加算準同型
c_sum = ((c0[0] + c1[0]) % q, (c0[1] + c1[1]) % q)
# 復号: 0 + 1 = 1 (mod 2)、ただし Regev は ℤ_2 平文なので 1
print(decrypt(sk, c_sum))  # → 1
```

### 7.2 実行の注意

実際の暗号ライブラリは:
- ガウス分布の正確なサンプラー
- $q$ を NTT-friendly prime に
- Double-CRT 表現で高速化

上記は **教育目的** の簡略版。セキュリティには使えない。

## 8. Ring-LWE 版への発展

### 8.1 ベクトル → 多項式

LWE 版を Ring-LWE 版に書き換えるには、**ベクトル $\mathbf{a}, \mathbf{s}, \mathbf{e}$ を多項式 $a, s, e \in R_q$ に置き換える**。

### 8.2 Ring-LWE 版 Regev

```
KeyGen (Ring-LWE):
    s ← R_q (スパース係数)
    a ← R_q (一様)
    e ← χ_R
    b ← -a · s + e
    pk = (a, b), sk = s

Enc (Ring-LWE):
    u, e1, e2 ← χ_R
    c1 ← a · u + e1
    c2 ← b · u + e2 + ⌊q/2⌋ · m       # m ∈ R_2 (多項式平文)

Dec (Ring-LWE):
    v ← c1 · s + c2 = (ノイズ) + ⌊q/2⌋ · m
    m ← round(v · 2 / q) mod 2
```

### 8.3 サイズ比較

- LWE 版: 暗号文サイズ $O(n \log q)$ で平文 1 ビット
- Ring-LWE 版: 暗号文サイズ $O(n \log q)$ で平文 $n$ ビット

**$n$ 倍の効率**。これが Ring-LWE ベーススキームが主流になる理由。

### 8.4 多項式平文の符号化

平文を多項式 $m(X) = m_0 + m_1 X + \cdots + m_{n-1} X^{n-1}$ として扱う。各係数が独立な平文スロット。

- BGV: 係数を $\mathbb{Z}_t$ に制限
- BFV: 同様
- CKKS: 係数を複素数に（エンコーディング経由）

Article 11-12 で詳述。

## 9. Q&A

### Q1: Regev スキームは実用？

**そのままでは実用しない**。教育用・理論解析用。実用は BGV/BFV/CKKS/TFHE。ただし **IoT の軽量暗号** として Regev 系スキームが検討されることはある。

### Q2: 1 ビットしか暗号化できないの？

**Regev オリジナルは 1 ビット**。しかし複数ビット版（Regev-LWE、Gentry-Peikert-Vaikuntanathan）も存在。Ring-LWE 版ならネイティブで $n$ ビット。

### Q3: $\mathbf{r}$ が 0/1 でないと安全性に影響？

$\mathbf{r}$ を **小さな分布**（たとえばガウス）にしても安全性は保たれる。ただし 0/1 が最も計算効率が良い。

### Q4: なぜ $\mu \cdot \lfloor q/2 \rfloor$ を足すのか？

**最上位ビットに平文を「分離」して載せる**ため。$q/2$ は $\mathbb{Z}_q$ の「真ん中」なので、$0$ と $q/2$ はノイズで混同されにくい。

### Q5: ノイズが 0 だとどうなる？

**LWE が崩壊**して、ガウス消去で秘密鍵が簡単に求まる。ノイズは **必須** の安全要素。

### Q6: Regev 論文はなぜ重要？

**LWE 仮定の導入** と **格子問題への還元証明** が含まれており、現代の格子ベース暗号（FHE、署名、鍵交換）すべての理論的土台。2005 年の STOC・2009 年の JACM 版が引用数 1 万超の古典論文。

## 10. まとめ

### 本記事で学んだこと

- Regev 2005 の公開鍵暗号スキームを、**鍵生成・暗号化・復号** まで完全にトレース
- **復号の正当性**: $v = \langle \mathbf{e}, \mathbf{r}\rangle + \mu q/2$ で、ノイズ部分が $q/4$ 以下なら $\mu$ を分離できる
- **IND-CPA 安全性**: Decision-LWE 仮定から導かれる
- **準同型加算**: 暗号文を足すと平文の和の暗号文。ノイズは $\sqrt{L}$ 倍に増加
- **Python 擬似コード** で全体の動きが確認できる
- **Ring-LWE 版**: 多項式環に持ち上げることで $n$ 倍の効率

### 次の記事（Article 8）へ

次の記事では、**準同型乗算** をどう実現するかを扱う。Regev スキームでは素朴な乗算が暗号文のサイズを膨張させる問題があり、それを **再線形化 (relinearization)** で解決する。さらに **モジュラス切り替え (modulus switching)** でノイズを管理する技術を学ぶ。ここから FHE 固有の技術に入る。

### 3行サマリ

- **Regev スキーム**: 秘密鍵 $\mathbf{s}$、公開鍵 $(A, A\mathbf{s}+\mathbf{e})$、暗号文 $(\mathbf{c}_1, c_2)$
- **復号**: $c_2 - \langle \mathbf{c}_1, \mathbf{s}\rangle \approx \mu \cdot q/2$。ノイズ < $q/4$ なら成功
- **加算準同型**: 成分ごとの足し算で $\mu + \mu'$ の暗号文。乗算には別の工夫が必要

---

## 参考文献

- Oded Regev. *On Lattices, Learning with Errors, Random Linear Codes, and Cryptography.* STOC 2005. JACM 2009.
- Chris Peikert. *Public-Key Cryptosystems from the Worst-Case Shortest Vector Problem.* STOC 2009.
- Daniel Lowengrub. *Fully Homomorphic Encryption from Scratch.* 2024. [https://www.daniellowengrub.com/blog/2024/01/03/fully-homomorphic-encryption](https://www.daniellowengrub.com/blog/2024/01/03/fully-homomorphic-encryption)
