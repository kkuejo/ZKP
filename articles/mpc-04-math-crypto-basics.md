---
title: "[MPC入門シリーズ 4/15] 数学・暗号の基礎 — 有限体、多項式、PRF、コミットメント"
emoji: "🧰"
type: "tech"
topics:
  - mpc
  - cryptography
  - finitefield
  - prf
  - commitment
published: false
---

**日付**: 2026年4月24日
**学習内容**: 第5記事以降で展開する Shamir 秘密分散、BGW、Oblivious Transfer、Garbled Circuits を理解するための**数学・暗号のツールキット**をまとめる。具体的には (1) 有限体 $\mathbb{F}_p$ と剰余演算の復習、(2) 多項式とラグランジュ補間、(3) Pseudorandom Function (PRF) と Pseudorandom Generator (PRG)、(4) コミットメントスキーム、(5) Random Oracle モデル、(6) 計算量的安全と統計的安全の区別、の6つを押さえる。ZKPシリーズと重複する部分は要点だけ示し、**MPC 特有の使われ方**を強調する。

## 0. 本記事の位置づけ

MPC のプロトコルを理解するには、ある程度の数学と暗号基礎が必要だ。本記事はその **最低限** を 1 本にまとめるのが目的で、深い代数や証明は扱わない。

既に ZKP シリーズ(特に Article 6: 有限体)を読んだ読者は**斜め読み**で十分。初めての読者には「後の章で出てくる記号と用語を、最低限言えるようになる」ことを目標に読んでほしい。

本記事の構成:

- **第1章**: 有限体 $\mathbb{F}_p$(Shamir SS の土台)
- **第2章**: 多項式とラグランジュ補間
- **第3章**: PRF と PRG(GC と OT の中核)
- **第4章**: コミットメントスキーム
- **第5章**: Random Oracle とハッシュの理想化
- **第6章**: 安全性の尺度(Perfect, Statistical, Computational)
- **第7章**: Q&A

## 1. 有限体 $\mathbb{F}_p$ — MPC 計算の舞台

### 1.1 なぜ有限体か

MPC は「秘密を $n$ 個のシェアに分割し、演算後に合流する」ことが頻出。有限体 $\mathbb{F}_p$ を選ぶ理由:

- **加減乗除が閉じている**(シェアで演算できる)
- **表現が有限ビット**(整数のように無限桁にならない)
- **ランダム性が一様**(シェアを乱数で masking すると秘密が完全に隠れる)

整数 $\mathbb{Z}$ は加減乗算は閉じるが除算で閉じない。実数 $\mathbb{R}$ は無限ビット必要。$\mathbb{F}_p$ がスイートスポット。

### 1.2 定義

$p$ を素数とすると、$\mathbb{F}_p = \{0, 1, 2, \ldots, p-1\}$ は **素数位数の体**。加減乗除はすべて $\bmod p$ で行う:

$$
a + b \equiv (a + b) \pmod{p}
$$
$$
a \times b \equiv (a \times b) \pmod{p}
$$

**逆元** $a^{-1}$ は $a \times a^{-1} \equiv 1 \pmod{p}$ を満たす要素。$p$ が素数なら $0$ 以外のすべての $a$ に存在する。計算は拡張ユークリッド互除法か、フェルマーの小定理 $a^{-1} \equiv a^{p-2} \pmod{p}$ で求められる。

### 1.3 MPC で使われる典型的な $p$

| 用途 | 素数 $p$ のサイズ | 典型例 |
|---|---|---|
| Shamir SS (情報理論的) | 32〜64 bit | $p = 2^{31}-1$(Mersenne prime) |
| BGW (論文例) | 任意 | $p$ > プレイヤー数 |
| 楕円曲線ベース OT | 256 bit | Curve25519 の base field |
| SPDZ (BDOZ MAC) | 128 bit | $p \approx 2^{128}$ |
| 汎用 FHE との併用 | 数千 bit | BGV/BFV ring |

SS や BGW では、**$p$ はシェアの組 $\{1, 2, \ldots, n\}$ がすべて区別可能であればよい**ので、理論上は $p > n$ で十分。ただし攻撃者が秘密を当てる確率 $1/p$ を無視できるほど小さくするには $p \gtrsim 2^{64}$ が望ましい。

### 1.4 多項式 $\mathbb{F}_p[x]$

$\mathbb{F}_p$ 係数の多項式の集合 $\mathbb{F}_p[x]$ は、**環(ring)**。

$$
f(x) = a_0 + a_1 x + a_2 x^2 + \cdots + a_d x^d \quad (a_i \in \mathbb{F}_p)
$$

**事実**: $d$ 次以下の多項式は、**$d+1$ 個の異なる点 $(x_i, y_i)$ で一意に決まる**。これが Shamir SS の根幹。

## 2. ラグランジュ補間 — Shamir SS の数学的エンジン

### 2.1 補間問題

$(x_1, y_1), \ldots, (x_k, y_k)$ という $k$ 個の点(ただし $x_i$ はすべて異なる)があるとする。これを通る **高々 $k-1$ 次の多項式** $f(x)$ は一意に存在し、次の式で書ける:

$$
f(x) = \sum_{i=1}^{k} y_i \cdot \ell_i(x)
$$

ここで **ラグランジュ基底多項式** $\ell_i(x)$ は:

$$
\ell_i(x) = \prod_{j \neq i} \frac{x - x_j}{x_i - x_j}
$$

$\ell_i(x_j) = \delta_{ij}$(クロネッカーのデルタ)を満たすので、$f(x_j) = y_j$ が成立する。

### 2.2 具体例

2点 $(1, 3), (2, 5)$ を通る1次以下の多項式を求めたい:

$$
\ell_1(x) = \frac{x - 2}{1 - 2} = -(x - 2) = -x + 2
$$
$$
\ell_2(x) = \frac{x - 1}{2 - 1} = x - 1
$$

$$
f(x) = 3 \cdot (-x + 2) + 5 \cdot (x - 1) = -3x + 6 + 5x - 5 = 2x + 1
$$

確認: $f(1) = 3, f(2) = 5$ ✓

### 2.3 Shamir SS での使い方(プレビュー)

Article 5 で詳述するが、Shamir SS では:

1. Dealer が秘密 $s$ を定数項にもつ $t$ 次多項式 $f(x) = s + a_1 x + \cdots + a_t x^t$ をランダムに作る
2. $n$ 個のシェア $f(1), f(2), \ldots, f(n)$ を各プレイヤーに配る
3. $t+1$ 個のシェアを集めればラグランジュ補間で $f$ を復元、$s = f(0)$ を取得
4. $t$ 個以下のシェアからは $s$ について**何もわからない**(一様乱数に見える)

### 2.4 Python での実装例

```python
def lagrange_interpolate(points, x, p):
    """
    points: [(x_1, y_1), ..., (x_k, y_k)] のリスト
    x: 補間したい点
    p: 素数(有限体の位数)
    return: f(x) mod p
    """
    result = 0
    k = len(points)
    for i in range(k):
        xi, yi = points[i]
        # ラグランジュ基底 ell_i(x)
        num, den = 1, 1
        for j in range(k):
            if i == j:
                continue
            xj, _ = points[j]
            num = (num * (x - xj)) % p
            den = (den * (xi - xj)) % p
        # den の逆元(フェルマーの小定理)
        den_inv = pow(den, p - 2, p)
        result = (result + yi * num * den_inv) % p
    return result
```

第5記事で、これを使って実際に Shamir SS を動かしてみる。

## 3. PRF と PRG — 擬似ランダム性の2本柱

### 3.1 Pseudorandom Generator (PRG)

**定義**: $G: \{0,1\}^\kappa \to \{0,1\}^\ell$($\ell > \kappa$)。短いシード $s$ から長い擬似乱数列 $G(s)$ を生成。

**安全性**: 多項式時間の敵が $G(s)$ と**真のランダム列** $r \in_R \{0,1\}^\ell$ を区別できる確率は無視できる。

**MPC での使い方**:

- **GC の wire label 生成**: PRG で全ラベルをシードから派生
- **乱数の拡張**: 少ないシードから大量のガーブル化用乱数を派生
- **Cut-and-Choose**: 各 GC を異なるシードで決定論的に生成、検査時にシードを開示

### 3.2 Pseudorandom Function (PRF)

**定義**: $F: \{0,1\}^\kappa \times \{0,1\}^n \to \{0,1\}^m$。鍵 $k$ を固定した $F_k(\cdot)$ は、入力から出力への**擬似ランダム関数**のように見える。

**安全性**: 多項式時間の敵が、$F_k$ と**真のランダム関数**のオラクルを区別できる確率は無視できる。

**MPC での使い方**:

- **OT Extension (IKNP)**: PRF で長いメッセージを派生
- **Authenticated Garbling**: PRF で MAC を生成
- **Oblivious PRF (OPRF)**: Private Set Intersection で核心

**実装**: 実用では AES や HMAC-SHA256 が標準的な PRF 候補。

### 3.3 PRG と PRF の関係

- **PRG から PRF**: GGM 構成 (Goldreich-Goldwasser-Micali 1984) で、PRGを再帰的に適用して PRF を作れる
- **PRF から PRG**: 自明(入力を拡張する $F_k(0), F_k(1), \ldots$ を並べる)

理論的にはどちらも **OWF(一方向関数)の存在** から構成可能。実装では AES が両方を兼ねる。

### 3.4 Correlation Robust Hash Function

MPC で特によく使われる概念。**ハッシュ $H$ が correlation robust** とは、固定の $\Delta$ に対して $H(x \oplus \Delta)$ が PRF のように見える性質。

- **FreeXOR** の安全性証明で必須
- **OT Extension (IKNP)** で必須
- Random Oracle で代用されることも多い

Article 10 で FreeXOR を扱う際に具体例を示す。

## 4. コミットメントスキーム

### 4.1 定義

**コミットメント** $C = \mathsf{Commit}(m, r)$ は、メッセージ $m$ と乱数 $r$ からコミットメント $C$ を計算する関数。2つの性質を持つ:

**Hiding(隠蔽性)**: $C$ からは $m$ について何もわからない。

**Binding(束縛性)**: コミットした後で $m$ を変更できない。

直感: **電子封筒**。中身(メッセージ)が見えず(Hiding)、後から差し替えられない(Binding)。

### 4.2 2種類の安全性

| 安全性の組み合わせ | 例 |
|---|---|
| Perfect Hiding + Computational Binding | Pedersen commitment |
| Computational Hiding + Perfect Binding | Hash-based(RO model) |

**両方 Perfect は不可能**(情報理論的限界)。

### 4.3 代表的な構成

**Hash-based(RO model)**:

$$
C = H(m \| r)
$$

$H$ を Random Oracle として扱えば、Hiding は $r$ の長さ、Binding は衝突耐性から。実装は最も軽量。MPC でよく使う。

**Pedersen commitment**(離散対数ベース):

$$
C = g^m \cdot h^r \in G
$$

ここで $g, h$ は離散対数が未知の群の生成元。Perfect Hiding(任意の $m$ を $C$ に対応する $r$ がある) + Computational Binding(DLogを解けば偽造可)。

### 4.4 MPC での使い方

- **Cut-and-Choose**: GC のハッシュをコミットして、後で開く
- **Coin Tossing**: 両者が乱数をコミットし、後に開いて XOR する
- **入力コミットメント**: Malicious MPC で入力を固定する

## 5. Random Oracle モデル

### 5.1 定義

**Random Oracle** $H: \{0,1\}^* \to \{0,1\}^\kappa$ は、**入力ごとに完全に独立な一様乱数**を返す理想化されたハッシュ関数。実行中に入力がある度に、そのステートフルなオラクルが乱数を返す(同じ入力には同じ値)。

### 5.2 ROモデルの使い方

プロトコルの安全性を証明する際に、$H$ を RO として扱い、敵を RO へのクエリ回数で制限する。実用ではこれを **SHA-256** などで置き換える(これは heuristic。RO ≠ SHA-256 だが、実用上は安全とみなす)。

### 5.3 MPC で RO を使う場面

- **Garbled Circuit のガーブル化関数**: $\mathsf{GarbleGate}$ の内部で $H$ を使う
- **コミットメント**: $C = H(m \| r)$
- **Fiat-Shamir 変換**: 対話型プロトコルを非対話化
- **OT Extension**: IKNP の内部で correlation robust hash として

### 5.4 RO の限界

Canetti-Goldreich-Halevi (1998) による **Random Oracle Methodology のパラドックス**: RO で安全だが、**任意の具体ハッシュに置き換えると破れる**プロトコルが存在する(極めて人工的だが)。

実用的な結論: **RO を使った証明は heuristic だが、実際の attacks は出ていない**。使うが、standard model(RO 無し)の証明があればそちらを好む。

## 6. 安全性の尺度 — Perfect / Statistical / Computational

### 6.1 3段階

2つの確率変数 $X$ と $Y$ が「区別できない」と言うとき、3段階ある:

**Perfect (完全)**: $\Pr[X = v] = \Pr[Y = v]$ for all $v$。同じ分布。

**Statistical (統計的)**: 

$$
\Delta(X, Y) := \frac{1}{2} \sum_v |\Pr[X = v] - \Pr[Y = v]| \leq \nu(\kappa)
$$

統計的距離が**無視できる関数** $\nu$ 以下。計算無限の敵でも区別できない。

**Computational (計算量的)**: 多項式時間の敵 $D$ に対して、

$$
|\Pr[D(X) = 1] - \Pr[D(Y) = 1]| \leq \nu(\kappa)
$$

### 6.2 セキュリティパラメータ

- **Computational security parameter $\kappa$**: 通常 128 または 256。攻撃者のオフライン計算量を制限
- **Statistical security parameter $\sigma$**: 通常 40〜80。1回限りの確率的攻撃を制限

実用では「攻撃成功確率 $\leq 2^{-\sigma} + \nu(\kappa)$」と表現する。$\sigma$ は回数制限、$\kappa$ は計算量制限に対応する。

### 6.3 MPC での使い分け

- **Information-Theoretic MPC** (BGW, CCD): すべて Perfect または Statistical
- **Computational MPC** (GMW, Yao's GC, SPDZ): Computational
- **Hybrid**: OT や OT Extension は Computational だが、それを使う BGW はその上で Perfect

Article 13 の Cut-and-Choose では、$2^{-\sigma}$ の確率で攻撃者が検出を逃れる設計になる。$\sigma = 40$ を取ると 1 兆分の 1 のオーダーでしか騙せない。

## 7. Q&A — よくある疑問

### Q1: 体 $\mathbb{F}_{2^k}$ は使わないの?

**使う**。Boolean 回路ベースの MPC(Yao's GC, GMW Boolean)では本質的に $\mathbb{F}_2$ の拡大体。arithmetic MPC は主に $\mathbb{F}_p$($p$ 大きな素数)。両者を混在させる研究もある(ABY フレームワーク)。

### Q2: ラグランジュ補間はなぜ Shamir SS で動くの?

多項式は「$k-1$ 次まで」なら $k$ 点で一意決定。秘密は $f(0)$ として埋め込まれる。$k-1$ 点集めても、$f(0)$ は**どんな値にもなりうる**(残り1点の自由度で任意)。よって情報理論的に秘密が隠れる。第5記事で厳密に示す。

### Q3: PRF と暗号化の違いは?

- **PRF**: 鍵で関数を「固定」する(決定論的)
- **暗号化**: 乱数を使って同じ平文からも異なる暗号文を作れる(非決定論的、セマンティック安全)

ただし **PRF があれば暗号化は作れる** ($c = (r, F_k(r) \oplus m)$ の形)。

### Q4: RO を使う論文を見ると「in ROM」と書いてあるが?

**Random Oracle Model** の略。「RO 仮定下での証明」という意味。実装では実ハッシュに置き換えるので heuristic。ただし**そう明記することが誠実**(Standard Model と明確に区別)。

### Q5: Computational Security で 128 bit は永続的?

**量子コンピュータが実用化したら短くなる**。Grover's アルゴリズムで対称鍵は実効的に $\kappa/2$ に減る。**256 bit あれば量子後も 128 bit 相当の安全**。ただし MPC の多くの最新研究はまだ古典鍵 128 bit を想定。

### Q6: Pedersen Commitment の $g, h$ はどう選ぶ?

**$h$ の $g$ に対する離散対数 $\log_g h$ が未知**であるように選ぶ。実用では "nothing-up-my-sleeve"(通し番号、SHA の出力などから決定論的に生成)で透明性を確保。

### Q7: $\mathbb{F}_2^k$ vs $\mathbb{F}_{2^k}$ の違いは?

- $\mathbb{F}_2^k$: $\mathbb{F}_2$ を $k$ 個並べたベクトル空間(加算は各成分、乗算は定義されない)
- $\mathbb{F}_{2^k}$: $2^k$ 位数の体(加減乗除すべて定義)

MPC では両方出てくる。$\mathbb{F}_2^k$ は「ビット列」、$\mathbb{F}_{2^k}$ は「演算する世界」。

### Q8: MPC プロトコルの論文は数学的でハードルが高い。どう読めばいい?

**コツ**:

1. **プロトコル図を先に見る**(Figure 1 あたり)
2. **各メッセージが何を意味するか**(値なのか、コミットメントなのか、乱数なのか)
3. **シミュレータの構造**を読む(Real と Ideal の橋渡し)
4. 暗号仮定は最後に確認(DDH? LWE? RO?)
5. **関連する既存プロトコル**と比べて、何が新しいかを理解

Lindell の *How to Simulate It* (2017) は読み方の指南書として優秀。

## 8. まとめ

### 本記事で学んだこと

- **有限体 $\mathbb{F}_p$**: MPC 計算の舞台。加減乗除が閉じる世界
- **ラグランジュ補間**: $d$ 次多項式は $d+1$ 点で一意決定。Shamir SS の核心
- **PRG / PRF**: 擬似ランダム性の2本柱。GC の wire label 生成や OT Extension で必須
- **コミットメント**: Hiding + Binding の電子封筒。Malicious MPC で入力・中間値を固定
- **Random Oracle**: ハッシュの理想化。実装では SHA-256 で代用
- **安全性の3段階**: Perfect / Statistical / Computational。文献では $\approx_p, \approx_s, \approx_c$ で表記

### 次の記事(Article 5)へ

ここから実際のプロトコルに入る。**Article 5: Shamir 秘密分散**。

- $(t+1, n)$-threshold Secret Sharing の定義
- ラグランジュ補間を使った Share / Reconstruct
- Python での実装(コード付き)
- 情報理論的安全性の証明スケッチ
- 応用(閾値暗号、分散鍵生成の導入)

### 3行サマリ

- **有限体 $\mathbb{F}_p$ + ラグランジュ補間が Shamir SS/BGW の基礎**
- **PRF/PRG/コミットメント/RO が GC/OT/Malicious MPC の基礎**
- **安全性は Perfect < Statistical < Computational の3段階で議論される**

---

## 参考文献

- Jonathan Katz, Yehuda Lindell. *Introduction to Modern Cryptography*. CRC Press, 3rd ed. 2020. (特に Chapter 3–4: Pseudorandomness)
- Dan Boneh, Victor Shoup. *A Graduate Course in Applied Cryptography*. 2023. [https://toc.cryptobook.us/](https://toc.cryptobook.us/)
- Oded Goldreich. *Foundations of Cryptography: Volume 1 — Basic Tools*. Cambridge University Press, 2001.
- Mihir Bellare, Phillip Rogaway. *Random Oracles are Practical: A Paradigm for Designing Efficient Protocols*. ACM CCS 1993.
- Ran Canetti, Oded Goldreich, Shai Halevi. *The Random Oracle Methodology, Revisited*. STOC 1998.
- Torben Pryds Pedersen. *Non-Interactive and Information-Theoretic Secure Verifiable Secret Sharing*. CRYPTO 1991.
