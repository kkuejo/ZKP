# ゼロ知識証明とSNARK入門：完全解説

> **対象読者**: 数学・暗号理論の予備知識がない方でも理解できるよう、一から丁寧に解説します。

---

## 目次

1. [SNARKとは何か？](#1-snarkとは何か)
2. [なぜSNARKに商業的注目が集まるのか？](#2-なぜsnarkに商業的注目が集まるのか)
3. [ブロックチェーンへの応用](#3-ブロックチェーンへの応用)
4. [非ブロックチェーンへの応用例：偽情報対策](#4-非ブロックチェーンへの応用例偽情報対策)
5. [SNARKの数学的定義](#5-snarkの数学的定義)
6. [知識の健全性とは何か？](#6-知識の健全性とは何か)
7. [ゼロ知識とは何か？](#7-ゼロ知識とは何か)
8. [効率的なSNARKの構築方法](#8-効率的なsnarkの構築方法)
9. [コミットメントスキームとは？](#9-コミットメントスキームとは)
10. [多項式コミットメント](#10-多項式コミットメント)
11. [Poly-IOP：万能SNARKへの鍵](#11-poly-iop万能snarkへの鍵)
12. [SNARKの実践的な使い方](#12-snarkの実践的な使い方)
13. [まとめ：全体像の整理](#13-まとめ全体像の整理)

---

## 1. SNARKとは何か？

### 直感的な説明

**SNARK** は **S**uccinct **N**on-interactive **AR**gument of **K**nowledge の略で、日本語に訳すと「**簡潔な非対話型知識証明**」です。

噛み砕いて言うと：

> 「私はある秘密を知っています。でも、その秘密自体はあなたに教えたくない。でも、私が本当にその秘密を知っていることを証明したい。」

これを実現するのがSNARKです。

### 具体例で理解する

教科書では次の例が挙げられています：

$$\text{「私は } m \text{ を知っている。} SHA256(m) = 0 \text{ となる } m \text{ を。」}$$

- $SHA256$ はハッシュ関数（入力から固定長の出力を生成する関数）
- $m$ はその入力（メッセージ）

もし $m$ が 1GB のファイルであれば、「証拠として $m$ そのものを渡す」のは：
- 証拠が 1GB と巨大（**短くない**）
- 検証するのに全部読む必要があって時間がかかる（**速くない**）

SNARKはこれを解決します。

### SNARKの3つの性質

| 性質 | 意味 |
|------|------|
| **Succinct（簡潔）** | 証明が「短い」かつ「速く検証できる」 |
| **Non-interactive（非対話型）** | 証明者と検証者がリアルタイムでやり取りしなくてよい（証明書を一枚渡すだけ） |
| **Argument of Knowledge（知識証明）** | 証明者は本当にその「秘密（証人）」を知っている |

さらに **zk-SNARK** の「zk」は **Zero Knowledge（ゼロ知識）** を意味し：

> 証明 $\pi$ は $m$ について「何も明かさない」

つまり、証明を見ても秘密 $m$ の情報は一切漏れないのです。

---

## 2. なぜSNARKに商業的注目が集まるのか？

### 1991年の革命的アイデア

Babai・Fortnow・Levin・Szegedy（1991年）の論文「多対数時間での計算検証」より：

> 「このしくみを使えば、低速で高価なコンピュータ（L1ブロックチェーン）が、GPU群の計算結果を監視・検証できる」

当時は「一台の信頼できるPCが、スーパーコンピュータの群れの計算を監視できる」と書かれていましたが、これを現代に置き換えると：

- **スーパーコンピュータ** → GPUクラスタ（L2などのオフチェーン処理）
- **低速で高価なPC** → ブロックチェーン（L1）

ブロックチェーンは処理が遅くて高価ですが、SNARKを使えば「膨大な計算が正しく行われた」という事実を、小さな証明書一枚で確認できるのです。

### 商業的プレイヤー

StarkWare、Aztec、Matter Labs、Polygon、Scroll、RISC Zero、Aleo など多数の企業がSNARKを使ったプロダクトを構築しています。

---

## 3. ブロックチェーンへの応用

### アプリケーション I：計算のアウトソーシング（ゼロ知識不要）

**zkRollup（ゼロ知識ロールアップ）**：

1. オフチェーンのサービスが大量のトランザクションをまとめて処理する
2. 「正しく処理した」という簡潔な証明を L1 チェーンに提出する
3. L1 チェーンは証明書を素早く検証するだけでよい

**zkBridge（ゼロ知識ブリッジ）**：

- 異なるブロックチェーン間でのコンセンサス証明
- 資産を別チェーンに移転できるようになる

### アプリケーション II：プライバシーが必要な場合

**プライベートトランザクション**：

- 公開ブロックチェーン上でトランザクションの内容を隠す
- 例：Tornado Cash、Zcash、IronFish、Aleo

**コンプライアンス証明**：

- 「このプライベートなトランザクションは銀行法に準拠しています」という証明（Espresso）
- 「この取引所はゼロ知識で支払い能力がある」という証明（Raposa）

---

## 4. 非ブロックチェーンへの応用例：偽情報対策

### 問題の背景

ウクライナ紛争などでは、偽の写真・動画が大量に拡散されました。「この写真は本物か？」を検証する手段が必要です。

### C2PA：コンテンツ出所証明の標準規格

カメラ（例：ソニーが開発中）に秘密鍵 $sk$ を埋め込み、撮影時に：

- 写真データ
- 撮影場所・時刻などのメタデータ
- 署名 $\sigma = \text{Sign}(sk, \text{写真})$

をひとまとめにする（**C2PA形式**）。

閲覧者はこの署名を確認することで「本物のカメラで撮られた」と検証できます。

### 問題：加工後の写真はどうする？

新聞社は写真を公開前に加工します（リサイズ、クロップ、グレースケール変換など）。

加工後の写真には元の署名が通用しない → **検証できない！**

### SNARKによる解決

編集ソフトウェアが証明 $\pi$ を付与します：

> 「私は原本 $\text{Orig}$ と署名 $\text{Sig}$ の組を知っている。以下の条件をすべて満たす：」

1. $\text{Sig}$ は $\text{Orig}$ に対する正当な C2PA 署名
2. $\text{Photo}$ は $\text{Orig}$ に操作 $\text{Ops}$ を適用した結果
3. $\text{metadata}(\text{Photo}) = \text{metadata}(\text{Orig})$

閲覧者のPCは証明 $\pi$ を検証し、メタデータ（撮影場所・時刻）を表示するだけ。**原本 $\text{Orig}$ は秘密のまま**でよい。

### 驚異的なパフォーマンス

- **証明サイズ**：$\leq 1 \text{KB}$（極めて小さい！）
- **検証時間**：$\leq 10 \text{ms}$（ブラウザ上で！）
- **証明生成時間**：6000×4000 ピクセルの画像で数分（新聞社が一度だけ行う）

---

## 5. SNARKの数学的定義

### 算術回路とは？

まず、SNARKが扱う「問題」を数学的に表現するために**算術回路**を使います。

有限体 $\mathbb{F}_p = \{0, 1, \ldots, p-1\}$（$p$ は素数）を固定します。

**算術回路** $C: \mathbb{F}^n \rightarrow \mathbb{F}$ とは：
- 有向非巡回グラフ（DAG）
- 内部ノードのラベルは $+$、$-$、$\times$（演算）
- 入力ノードのラベルは $1, x_1, \ldots, x_n$

例えば $x_1(x_1 + x_2 + 1)(x_2 - 1)$ という多項式を表す回路が作れます。

$|C|$ = 回路のゲート数（回路の「大きさ」）

### 具体的な算術回路の例

$$C_{SHA}(h, m): SHA256(m) = h \text{ なら } 0 \text{ を出力、さもなくば非ゼロ}$$

実装すると約 20,000 ゲートが必要です。

$$C_{sig}(pk, m, \sigma): \sigma \text{ が } m \text{ に対する正当なECDSA署名なら } 0 \text{ を出力}$$

### 構造化回路と非構造化回路

| 種類 | 説明 |
|------|------|
| **非構造化回路** | 任意の配線を持つ回路（一般的） |
| **構造化回路** | 同じモジュール $M$ が繰り返される回路（一種の仮想マシン） |

構造化回路は「プロセッサの1ステップ $M$」を繰り返す形で、一部のSNARK技法はこちらにしか適用できません。

### NARK の定義

**NARK（非対話型知識証明）** とは、以下の三つ組 $(S, P, V)$ です：

公開算術回路 $C(\mathbf{x}, \mathbf{w}) \rightarrow \mathbb{F}$ に対して：

- $\mathbf{x} \in \mathbb{F}^n$：**公開ステートメント**（皆が知っている）
- $\mathbf{w} \in \mathbb{F}^m$：**秘密の証人（witness）**（証明者だけが知っている）

**前処理（セットアップ）**：

$$S(C) \rightarrow (pp, vp)$$

- $pp$：証明者用公開パラメータ（Prover Parameters）
- $vp$：検証者用公開パラメータ（Verifier Parameters）

**証明者 P**：

$$P(pp, \mathbf{x}, \mathbf{w}) \rightarrow \pi \quad \text{（証明書 } \pi \text{ を生成）}$$

**検証者 V**：

$$V(vp, \mathbf{x}, \pi) \rightarrow \text{accept or reject}$$

### NARK の要件

**完全性（Completeness）**：

$$\forall \mathbf{x}, \mathbf{w}: C(\mathbf{x}, \mathbf{w}) = 0 \Rightarrow \Pr[V(vp, x, P(pp, \mathbf{x}, \mathbf{w})) = \text{accept}] = 1$$

「正直な証明者は必ず受け入れられる」

**適応的知識健全性（Adaptive Knowledge Soundness）**：

$$V \text{ が受け入れる} \Rightarrow P \text{ は } C(\mathbf{x}, \mathbf{w}) = 0 \text{ となる } \mathbf{w} \text{ を「知っている」}$$

（抽出器 $E$ が $P$ から有効な $\mathbf{w}$ を取り出せる）

**（オプション）ゼロ知識**：

$$(C, pp, vp, \mathbf{x}, \pi) \text{ は } \mathbf{w} \text{ について「新たな情報を何も明かさない」}$$

### SNARK の定義（SNARKの「S」の意味）

NARKに「簡潔性」が加わったものがSNARKです：

**簡潔な NARK** = 証明のサイズと検証時間が「回路サイズより小さい」

$$\text{len}(\pi) = O_\lambda(\text{sublinear}(|\mathbf{w}|))$$

$$\text{time}(V) = O_\lambda(|\mathbf{x}|, \text{sublinear}(|C|))$$

**強い意味での簡潔性（Strongly Succinct）**：

$$\text{len}(\pi) = O_\lambda(\log(|C|))$$

$$\text{time}(V) = O_\lambda(|x|, \log(|C|))$$

$\log(|C|)$ は回路サイズの**対数**なので、極めて小さい！検証者は回路全体を読む時間すらない。

### なぜ自明な方法はダメか？

一番簡単な方法（自明なSNARK）：
1. 証明者が $\mathbf{w}$ を検証者に送る
2. 検証者が $C(\mathbf{x}, \mathbf{w}) = 0$ を自分で確認する

問題点：
1. $\mathbf{w}$ が長い場合、証明が「短くない」
2. $C(\mathbf{x}, \mathbf{w})$ の計算が重い場合、検証が「速くない」
3. $\mathbf{w}$ が秘密の場合、証明者は $\mathbf{w}$ を明かしたくない

---

## 6. 知識の健全性とは何か？

### 「知っている」とはどういう意味か？

> 証明者 $P$ が $\mathbf{w}$ を「知っている」とは、$\mathbf{w}$ が $P$ から「抽出できる」ことを意味する。

これは直感的には「証明者の内部状態を覗けば $\mathbf{w}$ が取り出せる」ということです。

### 形式的定義

$(S, P, V)$ が回路 $C$ に対して**適応的知識健全**であるとは：

すべての多項式時間敵対者 $A = (A_0, A_1)$ に対して：

$$gp \leftarrow S_{\text{init}}(), \quad (C, x, st) \leftarrow A_0(gp), \quad (pp, vp) \leftarrow S_{\text{index}}(C), \quad \pi \leftarrow A_1(pp, x, st)$$

のとき、

$$\Pr[V(vp, x, \pi) = \text{accept}] > 1/10^6 \quad \text{（無視できない確率で受け入れられる）}$$

ならば、効率的な**抽出器** $E$（$A$ を使う）が存在して：

$$w \leftarrow E(gp, C, x)$$

$$\Pr[C(x, w) = 0] > 1/10^6 - \varepsilon \quad \text{（$\varepsilon$ は無視できる誤差）}$$

つまり：「検証者を騙せる証明者がいれば、その証明者から有効な証人 $w$ を取り出せる機械が存在する」ということです。

---

## 7. ゼロ知識とは何か？

### ウォーリーを探せのたとえ

「ウォーリーはどこにいる？」という問題を考えます。

- 証明者：ウォーリーの場所を知っている
- 検証者：場所を教えてもらいたいが、証明者は「いる」ことだけ証明したい

**ゼロ知識な解決法**：証明者は巨大な黒い布を用意し、ウォーリーの部分だけ小さな穴を開けて見せる。検証者は「ウォーリーがいる」ことは確認できるが、「どこにいるか」はわからない。

### SNARKにおけるゼロ知識

$(C, pp, vp, \mathbf{x}, \pi)$ から $\mathbf{w}$ について「新しい情報が何も得られない」ことを意味します。

数学的には「シミュレーターが存在する」という形で定義されますが、ここでは直感的理解に留めます。

---

## 8. 効率的なSNARKの構築方法

### 前処理セットアップの種類

セットアップ $S(C; r) \rightarrow (pp, vp)$ の $r$ はランダムビットです：

| 種類 | 説明 | セキュリティ |
|------|------|------|
| **回路ごとの信頼セットアップ** | ランダム $r$ を各回路ごとに生成、秘密にする | 証明者が $r$ を知ると偽証明を作れる |
| **普遍的（更新可能）な信頼セットアップ** | $r$ は回路 $C$ に依存しない | ワンタイムのセットアップで多数の回路に使える |
| **透明なセットアップ** | 秘密データを使わない（信頼セットアップ不要） | 最も安全 |

$$S = (S_{\text{init}}, S_{\text{index}}): \quad S_{\text{init}}(\lambda; r) \rightarrow gp, \quad S_{\text{index}}(gp, C) \rightarrow (pp, vp)$$

$S_{\text{init}}$ はワンタイム・秘密 $r$ が必要。$S_{\text{index}}$ は決定論的アルゴリズム。

### 現在の主要SNARKの比較

| プロトコル | 証明者時間 | セットアップ | 耐量子性 |
|------------|-----------|------------|---------|
| **Groth'16** | ほぼ線形 $O(\|C\|)$ | 回路ごとの信頼 | なし |
| **Plonk / Marlin** | ほぼ線形 $O(\|C\|)$ | 普遍的信頼 | なし |
| **Bulletproofs** | ほぼ線形 $O(\|C\|)$ | 透明 | なし |
| **STARK** | ほぼ線形 $O(\|C\|)$ | 透明 | あり |

すべてのプロトコルで「**証明者の時間は $|C|$ にほぼ線形**」というのが大きな進歩です。

（$|C| \approx 2^{20}$ ゲートの場合での比較）

### SNARKの構築パラダイム

効率的なSNARKは以下の2要素を組み合わせて構築します：

```
(1) 関数コミットメントスキーム（暗号的オブジェクト）
               +
(2) 対応する対話型オラクル証明（IOP）（情報理論的オブジェクト）
               ↓
        汎用回路向けSNARK
```

---

## 9. コミットメントスキームとは？

### 直感的説明

コミットメントスキームは「封筒に手紙を入れて封をする」操作のようなものです：

1. **コミット**：メッセージ $m$ を封筒に入れて封をする → $\text{com}$
2. **公開（オープン）**：後で封筒を開けて $m$ を見せる

### 形式的定義

2つのアルゴリズム：

$$\text{commit}(m, r) \rightarrow \mathbf{com} \quad \text{（} r \text{ はランダム）}$$

$$\text{verify}(m, \mathbf{com}, r) \rightarrow \text{accept or reject}$$

**性質（非公式）**：

- **束縛性（Binding）**：$\mathbf{com}$ に対して二つの異なる有効なオープンを作れない（封筒の中身を後から変えられない）
- **隠蔽性（Hiding）**：$\mathbf{com}$ はコミットされたデータについて何も明かさない（封筒の外から中身がわからない）

### 標準的な構成

ハッシュ関数 $H: \mathcal{M} \times \mathcal{R} \rightarrow T$ を固定して：

$$\text{commit}(m, r) : \quad \mathbf{com} := H(m, r)$$

$$\text{verify}(m, \mathbf{com}, r) : \quad \mathbf{com} = H(m, r) \text{ なら accept}$$

適切なハッシュ関数 $H$ に対して隠蔽性と束縛性が成立します。

---

## 10. 多項式コミットメント

### 関数へのコミットメント

**関数のファミリー** $\mathcal{F} = \{f: X \rightarrow Y\}$ に対するコミットメントとは：

証明者が $f \in \mathcal{F}$ にコミットし、後で：

$$f(x) = y \quad \text{かつ} \quad f \in \mathcal{F}$$

を証明できることです。

**プロトコルの流れ**：

```
証明者                              検証者
f ∈ F を選ぶ
r ←$ R
com_f ← commit(f, r) ──── com_f ──→
                    ←─── x ∈ X ───
y, π（f(x)=y の証明）─── y, π ──→  accept/reject
```

### 4種類の重要な関数コミットメント

**1. 多項式コミットメント（Polynomial Commitment）**：

$$f(X) \in \mathbb{F}_p^{(\leq d)}[X] \quad \text{（次数 } \leq d \text{ の一変数多項式）}$$

**2. 多線形コミットメント（Multilinear Commitment）**：

$$f \in \mathbb{F}_p^{(\leq 1)}[X_1, \ldots, X_k] \quad \text{（各変数の次数が1以下の多変数多項式）}$$

例：$f(x_1, \ldots, x_k) = x_1 x_3 + x_1 x_4 x_5 + x_7$

**3. ベクトルコミットメント（Vector Commitment）**（例：マークルツリー）：

$$\vec{u} = (u_1, \ldots, u_d) \in \mathbb{F}_p^d \text{ にコミット}$$
$$f_{\vec{u}}(i) = u_i \text{ （任意のセルを開ける）}$$

**4. 内積コミットメント（Inner Product Commitment）**：

$$\vec{u} \in \mathbb{F}_p^d \text{ にコミット}$$
$$f_{\vec{u}}(\vec{v}) = \langle \vec{u}, \vec{v} \rangle = \sum_i u_i v_i \text{ （内積を証明）}$$

### 多項式コミットメントの詳細

証明者は $f(X) \in \mathbb{F}_p^{(\leq d)}[X]$ にコミットします。

**eval**（評価）プロトコル：公開 $u, v \in \mathbb{F}_p$ に対して：

$$f(u) = v \quad \text{かつ} \quad \deg(f) \leq d$$

を証明できます。検証者は $(d, \text{com}_f, u, v)$ を持ちます。

評価証明のサイズと検証時間は：

$$O_\lambda(\log d)$$

### なぜ自明な多項式コミットメントはダメか？

$$\text{commit}\!\left(f = \sum_{i=0}^d a_i X^i, r\right): \quad \text{com}_f \leftarrow H\!\left((a_0, \ldots, a_d), r\right)$$

eval の証明として $\pi = ((a_0, \ldots, a_d), r)$ を送れば検証できますが：

**問題**：証明 $\pi$ のサイズと検証時間が $d$ に**線形**（大きすぎる！）

### 鍵となる数学的観察：Schwartz-Zippel の補題

$$\text{零でない } f \in \mathbb{F}_p^{(\leq d)}[X] \text{ に対して、} r \overset{\$}{\leftarrow} \mathbb{F}_p \text{ をランダムに選ぶと：}$$

$$\Pr[f(r) = 0] \leq \frac{d}{p}$$

$p \approx 2^{256}$、$d \leq 2^{40}$ なら $d/p$ は無視できる（$\approx 2^{-216}$）。

**重要な帰結**：

$$r \overset{\$}{\leftarrow} \mathbb{F}_p \text{ のとき、} f(r) = 0 \text{ なら、ほぼ確実に } f \equiv 0 \text{（零多項式）}$$

→ **コミットされた多項式の零テスト**が可能！

さらに：

$$f, g \in \mathbb{F}_p^{(\leq d)}[X] \text{ に対して、}$$
$$r \overset{\$}{\leftarrow} \mathbb{F}_p \text{ のとき、} f(r) = g(r) \Rightarrow f = g \quad \text{（ほぼ確実）}$$

→ **コミットされた2つの多項式が等しいかの検査**が可能！

### 等価性検査プロトコル

```
証明者（f, g を知っている）              検証者（f, g のコミットを持つ）
                          ←── r ←$ F_p
y  ← f(r),  y' ← g(r)
π_f（f(r)=y の証明）     ─── y, π_f, y', π_g ──→  π_f, π_g が正当かつ y = y' なら accept
π_g（g(r)=y' の証明）
```

### Fiat-Shamir変換：対話型 → 非対話型（SNARK化）

上記の対話型プロトコルをSNARKにするには、検証者のランダム性を**ハッシュ関数**で置き換えます：

$$r \leftarrow H(x)$$

証明者が自分でランダム値 $r$ を生成し、検証者はそれを再現できます（$H$ は SHA256 などを使用）。

**定理**：$d/p$ が無視できる（Schwartz-Zippelが成立）かつ $H$ がランダムオラクルとしてモデル化されるなら、これは SNARKです。

### 多項式コミットメントの主要な構成法

| 構成法 | 前提 | 特徴 |
|--------|------|------|
| **KZG'10** | 双線形群（Bilinear groups） | 信頼セットアップ必要 |
| **Dory'20** | 双線形群 | 信頼セットアップ不要 |
| **FRI ベース** | ハッシュ関数のみ | eval 証明が長い |
| **Bulletproofs** | 楕円曲線 | 短い証明、検証時間 $O(d)$ |
| **Dark'20** | 位数不明群 | 透明セットアップ |

---

## 11. Poly-IOP：万能SNARKへの鍵

### F-IOP とは？

**$\mathcal{F}$-IOP（関数対話型オラクル証明）** は、関数コミットメントを使って一般回路向けSNARKを構築するための情報理論的プロトコルです。

算術回路 $C(x, w)$ と公開入力 $x \in \mathbb{F}_p^n$ に対して、$\exists w: C(x, w) = 0$ を以下のように証明します：

**セットアップ**：

$$\text{Setup}(C) \rightarrow pp, \quad vp = \left(\boxed{f_0}, \boxed{f_{-1}}, \ldots, \boxed{f_{-s}}\right)$$

$vp$ の中の $\boxed{f_i}$ は $\mathcal{F}$ に属する関数への**オラクル**（任意の点で評価できる関数）。

### Poly-IOP のプロトコルフロー

```
証明者 P(pp, x, w)                          検証者 V(vp, x)

  oracle f_1 ∈ F  ────────────────────────→    r_1 ←$ F_p
                   ←────────────── r_1 ─────
  oracle f_2 ∈ F  ────────────────────────→
                   ⋮
                   ←─────────── r_{t-1} ────    r_{t-1} ←$ F_p
  oracle f_t ∈ F  ────────────────────────→
                                                verify^{f_{-s},...,f_t}(x, r_1, ..., r_{t-1})
                                                （各 f_i を任意の点で評価できる）
```

**IOP の性質**：
- **完全性**：$\exists w: C(x, w) = 0$ ならば $\Pr[\text{検証者が accept}] = 1$
- **（無条件）知識健全性**：情報理論的に成立（暗号仮定不要！）
- **（オプション）ゼロ知識**：zk-SNARK にする場合

### Poly-IOP の具体例（少し作為的な例）

回路 $C(X, W) = 0 \Leftrightarrow X \subseteq W \subseteq \mathbb{F}_p$（集合の包含関係）

```
証明者 P(pp, X, W)                           検証者 V(vp, X)

f(Z) := ∏_{w∈W}(Z-w)
g(Z) := ∏_{x∈X}(Z-x)              f, q を送る →    g(Z) := ∏_{x∈X}(Z-x)
q(Z) := f/g ∈ F_p^{(≤d)}[X]                         r ←$ F_p
                                  ←── r ───
  ⇒ g(Z) は f(Z) の因数                              (i) w ← f(r) を評価
                                                    (ii) q' ← q(r) を評価
                                                    (iii) x ← g(r) を計算
                                                    (iv) x · q' = w なら accept
```

**知識健全性**：検証者が accept $\Rightarrow$ ほぼ確実に $f = g \cdot q \Rightarrow X \subseteq W$

**抽出器** $E(X, f, q, r)$：$f(Z)$ のすべての根を計算して証人 $W$ を出力

### SNARKの「動物園」（IOP の種類）

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Poly-IOP      Multilinear-IOP    Vector-IOP            │
│  (Sonic,       (Spartan,          (STARK,               │
│   Marlin,       Clover,            Breakdown,           │
│   Plonk, …)     Hyperplonk, …)     Orion, …)            │
│      ↓                ↓                 ↓               │
│  Poly-Commit  Multilinear-Commit    Merkle              │
│      ↓                ↓                 ↓               │
│      └────────────────┴─────────────────┘               │
│                        ↓                                │
│                Fiat-Shamir 変換（非対話型化）             │
│                        ↓                                │
│                    🔒 SNARK 🔒                           │
└─────────────────────────────────────────────────────────┘
```

---

## 12. SNARKの実践的な使い方

### 実際のシステム構成

```
DSLプログラム             SNARK対応形式              SNARK バックエンド
┌─────────────┐           ┌─────────────┐           ┌─────────────┐
│ Circom      │           │ 回路        │           │             │
│ ZoKrates    │  コンパイラ │ R1CS        │  →  証明者  │ 重い計算    │ → π
│ Leo         │ ─────────→│ EVM バイト  │           │             │
│ Zinc        │           │ コード      │           └─────────────┘
│ Cairo       │           │ ...         │                  ↑
│ Noir        │           └─────────────┘              x と証人
│ ...         │
└─────────────┘
```

**DSL（ドメイン固有言語）**：プログラマーが書きやすい言語（Circom、ZoKratesなど）

**コンパイル**：DSLプログラム → SNARK対応形式（R1CS など）

**SNARK バックエンド**：実際に証明 $\pi$ を生成するエンジン（重い計算だが一度だけ）

---

## 13. まとめ：全体像の整理

### zk-SNARK の全体像

```
zk-SNARK = SNARK + Zero Knowledge

SNARK = NARK + Succinct
       = Complete + Knowledge Sound + Succinct

Succinct: len(π) = O(log|C|), time(V) = O(|x|, log|C|)
```

### 重要概念の整理表

| 概念 | 意味 | たとえ |
|------|------|------|
| **証人 $\mathbf{w}$** | 秘密の知識 | 金庫の暗証番号 |
| **公開ステートメント $\mathbf{x}$** | 皆が知る情報 | 金庫の型番 |
| **算術回路 $C$** | 検証ルール | 「正しい暗証番号かチェックする機械」 |
| **証明 $\pi$** | 知識の証拠 | 「暗証番号を知っている」という書類 |
| **完全性** | 正直者は必ず証明できる | |
| **知識健全性** | 証明できるなら本当に知っている | |
| **ゼロ知識** | 証明から秘密はバレない | |
| **簡潔性** | 証明が小さく、検証が速い | |

### SNARKが実現すること

$$\text{1KBの証明書一枚で「1GBのデータを処理した」を数ミリ秒で検証できる}$$

これは、現代の暗号技術における最も重要な革新の一つです。

---

## 補足：数式まとめ

### Schwartz-Zippel の補題

$$\forall f \in \mathbb{F}_p^{(\leq d)}[X],\ f \neq 0: \quad \Pr_{r \overset{\$}{\leftarrow} \mathbb{F}_p}[f(r) = 0] \leq \frac{d}{p}$$

### SNARKの簡潔性条件

$$\text{len}(\pi) = O_\lambda(\log |C|)$$
$$\text{time}(V) = O_\lambda(|x|, \log |C|)$$

### コミットメント

$$\text{commit}(m, r) = H(m, r)$$
$$\text{verify}(m, \mathbf{com}, r) = \mathbf{1}[\mathbf{com} = H(m, r)]$$

### NARK の完全性

$$\forall \mathbf{x}, \mathbf{w}:\ C(\mathbf{x}, \mathbf{w}) = 0 \Rightarrow \Pr[V(vp, x, P(pp, \mathbf{x}, \mathbf{w})) = \text{accept}] = 1$$

---

*本資料は ZKP MOOC Lecture 2「Introduction to Modern SNARKs」（Dan Boneh, Shafi Goldwasser, Dawn Song, Justin Thaler, Yupeng Zhang）に基づいて作成されました。*
