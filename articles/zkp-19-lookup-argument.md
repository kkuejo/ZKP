---
title: "[ゼロ知識証明入門シリーズ 19/33] Lookup Argument と Custom Gate — 回路を劇的に軽くする"
emoji: "🔍"
type: "tech"
topics:
  - zkp
  - snark
  - lookup
  - plonkish
  - plookup
published: false
---

**日付**: 2026年4月22日
**学習内容**: 現代 SNARK（Plonkish, Halo2）で最も重要な拡張が **Lookup Argument** と **Custom Gate**。lookup は「$x$ が事前定義されたテーブル $T$ に入っている」ことを低コストで示す仕組みで、**range check（範囲検証）・ビット演算・ハッシュ関数 の制約数を桁違いに削減**する。本記事では **(1) lookup が解決する問題**、**(2) Plookup プロトコル**、**(3) Halo2 の lookup**、**(4) Log-derivative lookup (Haböck, 2022)**、**(5) Custom gate の設計**、**(6) 典型的なカスタムゲート（Poseidon, range check, XOR）**、**(7) Lookup + Custom Gate の組み合わせ効果** を見ていく。

## 0. 本記事の位置づけ

Article 16 で PLONK の基本形を見た。しかし基本形だけでは以下のような計算が重い:

- **範囲検証**: 「$x \in [0, 2^{32})$」を示すのに 32 制約
- **ビット演算**: $a \oplus b$ を 1 ビットごとに計算
- **ハッシュ関数**: SHA-256 で 27,000 制約

これらは「**事前に答えを知っているテーブル**」を使えば劇的に軽くなる。例: 「$(a, b, a \oplus b)$ の全組み合わせをテーブルに並べ、witness がその 1 行である」と示す。これが Lookup の発想。

構成:

- **第1章**: Lookup の動機
- **第2章**: Plookup プロトコル
- **第3章**: Halo2 の lookup
- **第4章**: Log-derivative lookup
- **第5章**: Custom gate の設計
- **第6章**: 典型的なカスタムゲート
- **第7章**: 組み合わせと実用効果
- **第8章**: Q&A とまとめ

## 1. Lookup の動機

### 1.1 Range Check の困難

「$x \in [0, 2^{32})$」を示す素朴な方法は**ビット分解**:

$$
x = \sum_{i=0}^{31} b_i \cdot 2^i, \quad b_i \in \{0, 1\}
$$

32 個のブール制約 + 1 線形制約 = 33 制約。

**Lookup 版**:

- テーブル $T = \{0, 1, 2, \ldots, 2^{32} - 1\}$
- 「$x \in T$」を示す = 1 制約

**33 倍の削減**。

### 1.2 XOR の困難

$a, b \in \{0, \ldots, 255\}$ の XOR $a \oplus b$ を計算するには、素朴にはビット分解 + 各ビット XOR = 約 25 制約。

**Lookup 版**:

- テーブル $T = \{(a, b, a \oplus b) : a, b \in [0, 256)\}$（65,536 行）
- 「$(a, b, c) \in T$」を示す = 1 制約

### 1.3 ZK-friendly ハッシュとの組み合わせ

Poseidon の S-box ($x^5$) などは lookup で 1 制約化できる:

- テーブル $\{(x, x^5) : x \in \mathbb{F}_p\}$

ただし $\mathbb{F}_p$ 全体は巨大なので、実際には部分領域に限定。

### 1.4 Lookup の効果

Halo2 の Zcash Orchard では、lookup を駆使してシールデッドトランザクション生成を劇的に高速化。

## 2. Plookup プロトコル

### 2.1 Plookup (Gabizon, Williamson, 2020)

PLONK を lookup 対応にした最初のプロトコル。

### 2.2 問題設定

テーブル $T = \{t_1, t_2, \ldots, t_N\}$、主張する witness 列 $F = \{f_1, \ldots, f_n\}$ ($n \leq N$)。

**主張**: 各 $f_i \in T$。

### 2.3 Plookup の核アイデア

「$F$ のソート列」と「$T$ のソート列」を連結した新列 $s = \text{sort}(F \cup T)$ を作る。

**重要な性質**:

- $F \subseteq T$ ⟺ $s$ の隣接差は 0 または「$T$ の隣接差」

### 2.4 ランダム化等式

チャレンジ $\beta, \gamma$ を使って、グラフ product 等式:

$$
\prod_{i} (1 + \beta)(\gamma + f_i) \prod_{j} (\gamma(1 + \beta) + t_j + \beta t_{j+1})
$$

$$
= (1 + \beta)^n \prod_{k} (\gamma(1 + \beta) + s_k + \beta s_{k+1})
$$

これが成立 ⟺ $F \subseteq T$（Schwartz-Zippel ベースの argument）。

### 2.5 Grand Product 多項式

PLONK の置換引数と同じ形式で、累積積 $Z(X)$ を作り:

$$
Z(\omega^{i+1}) = Z(\omega^i) \cdot \frac{\text{左側の項}}{\text{右側の項}}
$$

$Z(\omega^n) = 1$ で閉じれば成立。

### 2.6 Prover コスト

- ソート: $O(n \log n)$
- 各ステップの計算: $O(n)$
- KZG コミット: $O(n)$

合計 $O(n \log n)$。重くない。

## 3. Halo2 の lookup

### 3.1 Halo2 Lookup

Halo2 は Plookup の改良版を実装。特徴:

- **複数のテーブル**をサポート
- **カラム単位**で lookup を指定
- **選択的 lookup**: 特定の行だけ lookup 対象

### 3.2 実用例: Zcash Orchard

Orchard のシールデッドノート検証では、複数の小テーブルを同時使用:

- 256-bit 数の 8-bit 分解
- Poseidon S-box の事前計算
- 楕円曲線点の倍算テーブル

これらすべてを lookup で高速化。

### 3.3 Halo2 の API

```rust
// 簡略化した疑似コード
meta.lookup(|meta| {
    let x = meta.query_advice(x_column, Rotation::cur());
    let t = meta.query_fixed(table_column);
    vec![(x, t)]
});
```

Rust で表現力豊かに lookup を宣言できる。

## 4. Log-derivative lookup (Haböck, 2022)

### 4.1 新アイデア

Plookup より簡潔で効率的な lookup プロトコル。log-derivative で書く:

$$
\sum_i \frac{1}{X - f_i} = \sum_j \frac{m_j}{X - t_j}
$$

ここで $m_j = |\{i : f_i = t_j\}|$（テーブル要素 $t_j$ が witness に何回現れるか）。

### 4.2 なぜ log-derivative か

$\log \prod_i (X - f_i) = \sum_i \log(X - f_i)$。微分すると $\sum_i \frac{1}{X - f_i}$。これを直接計算する。

### 4.3 利点

- **より短い証明**
- **より少ない Prover 作業**
- **多重集合 (multiset)** を自然に扱える

Plonky2, zkSync Era が採用。

### 4.4 BLK (Building-blocks Lookup)

さらに発展した Logup-GKR などでは、GKR プロトコル + log-derivative lookup で Prover が線形時間。

## 5. Custom Gate の設計

### 5.1 Custom Gate とは

PLONK の基本ゲート $q_L a + q_R b + q_M ab + q_O c + q_C = 0$ を拡張し、**任意の多項式制約**を 1 ゲートで表現。

### 5.2 一般形

$$
q_{\text{custom}}(X) \cdot P(a, b, c, d, \ldots, \text{回転入力}) = 0
$$

$P$ は多項式。回転入力 ($a_{\text{next}}, a_{\text{prev}}$) も使える → 複数行にまたがる制約。

### 5.3 例1: 楕円曲線加算

$(x_1, y_1) + (x_2, y_2) = (x_3, y_3)$ を 1 ゲートで:

$$
q_{\text{EC}} \cdot [(x_2 - x_1)^2 (x_3 + x_1 + x_2) - (y_2 - y_1)^2] = 0
$$

通常なら 10+ 制約が、1 カスタムゲートで済む。

### 5.4 例2: Poseidon S-box

$c = a^5$ を 1 ゲートで:

$$
q_{\text{sbox}} \cdot (a^5 - c) = 0
$$

多項式の次数は 5 なので、ゲート次数の上限にひっかからない範囲で許容。

### 5.5 次数の制約

カスタムゲートの多項式次数が大きいと Prover コストが増加。Halo2 では**次数 5〜9 程度**が実用範囲。

## 6. 典型的なカスタムゲート

### 6.1 Range Check Gate

特定範囲での範囲検証を 1 ゲートで:

$$
q_{\text{range}} \cdot \prod_{k=0}^{3} (a - k) = 0 \iff a \in \{0, 1, 2, 3\}
$$

4 項の積で 2 ビット範囲。8 ビットなら 256 項で次数が高くなりすぎるので lookup を併用。

### 6.2 Bit Decomposition Gate

$a = 2 b + c$ で $c \in \{0, 1\}$ を保証:

$$
q_{\text{bitdec}} \cdot [c (c - 1)] = 0
$$

$$
q_{\text{bitdec}} \cdot [a - 2 b - c] = 0
$$

同じ行で 2 制約。

### 6.3 XOR Gate

$c = a \oplus b$ を、bit-decomposition + lookup:

- $a$ のビット列 $a_0, \ldots, a_{n-1}$
- $b$ のビット列 $b_0, \ldots, b_{n-1}$
- 各 $i$ で $c_i = a_i + b_i - 2 a_i b_i$
- 全体を $c = \sum c_i 2^i$ で合成

8-bit なら 8 ビットカスタムゲート + 1 合成ゲート。

### 6.4 Poseidon Round Gate

Poseidon の 1 ラウンドをまるごと 1 行で:

$$
q_{\text{poseidon}} \cdot \big[ \text{round\_output} - \text{MixLayer}(\text{SBox}(\text{round\_input} + \text{roundConstants})) \big] = 0
$$

ただし Round 内部は多段なので、複数行に分けることもある。

### 6.5 Keccak Gate

Keccak (SHA-3) は SNARK で**超重い**。カスタムゲート + lookup で:

- $\chi$ ステップ: lookup
- $\theta$ ステップ: XOR テーブル
- Step 1 ゲートで Keccak 1 ラウンドを表現

Plonky2 は Keccak 1 ラウンドを約 100 制約で実装。

## 7. 組み合わせと実用効果

### 7.1 SHA-256 の制約数比較

| 実装方法 | 制約数（1 ブロック） |
|---|---|
| 素朴な R1CS | 27,000 |
| PLONK + lookup | 1,000 |
| PLONK + lookup + custom gate | 300 |

### 7.2 Poseidon の制約数

| 実装方法 | 制約数（1 permutation） |
|---|---|
| 素朴な R1CS | 2,000 |
| PLONK | 500 |
| Halo2 + lookup | 200 |

### 7.3 ECDSA 検証

| 実装方法 | 制約数 |
|---|---|
| R1CS | 100,000 |
| Halo2 | 15,000 |
| Plonky2 + FRI | 8,000 |

### 7.4 zkEVM の恩恵

zkEVM は EVM 命令を 1 つずつ回路化。各 opcode に custom gate + lookup を割り当てることで、1 命令あたりの制約を 100 程度に抑えられる。

## 8. Q&A

### Q1: Lookup と Permutation Argument の関係は？

両方とも「**ランダム線形結合**」+「**grand product**」を使う類似の構造。Plookup は Permutation を多重集合包含に一般化した形。

### Q2: テーブルサイズ $N$ が大きいとコスト増？

**Prover 側**: ソートと累積積で $O(N + n)$。$N$ が巨大だと Prover メモリが膨らむ。

**Verifier 側**: $O(1)$ 程度（コミットメントのみ）。

現実的には $N \leq 2^{20}$ 程度。

### Q3: Halo2 の lookup は Plookup と違う？

- Plookup: 1 組のテーブルと witness 集合
- Halo2: **複数の独立したテーブル**、かつ lookup 先に **式** も指定可能

### Q4: Custom Gate を作るコスト？

- 回路設計者の作業: 高次多項式の式組み
- Prover コスト: 次数が上がると FFT コストも上がる
- 実用では**頻繁に使われるパターン**にのみ custom gate を作るべき

### Q5: Dynamic lookup (witness 依存テーブル) は？

- 通常の lookup: 静的なテーブル
- Dynamic lookup: witness に応じてテーブルが変わる
- Halo2 や最新プロトコルで対応

実装は複雑だが、表現力が上がる。

### Q6: Lookup なしで書けない計算は？

理論的には**すべて Lookup なしで書ける**（R1CS で十分）。ただしコストが大きすぎて実用的でない計算（SHA-256 を 100 万回など）は lookup + custom gate に頼る。

## 9. まとめ

### 本記事の要点

1. **Lookup Argument** は「witness がテーブルに入る」を低コストで示す
2. **Plookup**: Plonkish lookup の元祖。ソート + grand product
3. **Halo2 Lookup**: 複数テーブル・選択的 lookup で柔軟
4. **Log-derivative Lookup**: より短く効率的（Plonky2 で採用）
5. **Custom Gate**: 任意の多項式制約を 1 ゲートで
6. Lookup + Custom Gate で SHA-256 が 100 倍軽くなる、zkEVM の基盤
7. 次数の制約（Halo2 で 5〜9）があるので、バランス設計が必要

### 次の記事（Article 20）へ

次の記事は **Recursive SNARK と Nova Folding**。SNARK の証明の中で、別の SNARK の検証回路を走らせる「再帰」の仕組みと、最近話題の folding scheme Nova を扱う。無限の計算を固定サイズの証明で示せる。

### 3行サマリ

- **Lookup = witness がテーブルに入ることを示す低コスト手法**
- **Custom Gate = 複雑な制約を 1 行で書く**
- **SHA-256 を 100 倍軽くする、zkEVM / Poseidon / ECDSA の必需品**

---

## 参考文献

- Ariel Gabizon, Zachary J. Williamson. *plookup: A simplified polynomial protocol for lookup tables.* ePrint 2020/315.
- Ulrich Haböck. *Multivariate Lookups Based on Logarithmic Derivatives.* ePrint 2022/1530.
- Electric Coin Company. *Halo2 Book: Lookup arguments.* 2023.
- Polygon Zero. *Plonky2 Design.* 2023.
- Scroll. *Lookup Arguments in Halo2.* Docs, 2024.
