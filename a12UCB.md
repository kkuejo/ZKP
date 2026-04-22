# zkEVM の設計・最適化・応用：完全解説

> **講義元**: Zero Knowledge Proofs MOOC（Dan Boneh, Shafi Goldwasser, Dawn Song, Justin Thaler, Yupeng Zhang）  
> **ゲスト講師**: Ye Zhang（Scroll）

---

## 目次

1. [背景と動機](#1-背景と動機)
2. [zkEVMをゼロから構築する](#2-zkevm-をゼロから構築する)
3. [面白い研究課題](#3-面白い研究課題)
4. [zkEVMのその他の応用](#4-zkevm-のその他の応用)

---

## 1. 背景と動機

### 1.1 Scroll（スクロール）とは何か？

**Scroll** は、Ethereum（イーサリアム）のスケーリング（処理能力向上）ソリューションです。  
より具体的には、**EVM同等（EVM-equivalent）のzk-Rollup（ゼロ知識ロールアップ）** です。

- **EVM同等** とは：既存のイーサリアムのスマートコントラクト（プログラム）をそのまま動かせるということ
- **zk-Rollup** とは：多数のトランザクション（取引）を一括で証明する技術（後述）

---

### 1.2 Layer 1 ブロックチェーンの仕組み

#### ブロックチェーンとは（超基本）

ブロックチェーンとは、世界中に散らばる多数のコンピュータ（ノード）が同じ取引履歴を持ち合う分散型台帳です。

**Layer 1（L1）のフロー：**

1. ユーザーが取引（$TX_1, TX_2, \ldots, TX_n$）を送信
2. 全ノードがその取引を検証（正しいか・不正でないか）
3. 合意（コンセンサス）が取れたらブロックに追加
4. ブロックが連なってチェーンになる

**Layer 1の構成要素（EVMの内部）：**

| コンポーネント | 役割 |
|---|---|
| EVM（Ethereum Virtual Machine） | 命令（opcode）を実行する仮想コンピュータ |
| State Machine（状態機械） | 現在の状態を管理する |
| Stack（スタック） | 計算の一時記憶領域（後入れ先出し） |
| Memory（メモリ） | 実行中のデータ保存 |
| Storage（ストレージ） | 永続的なデータ保存（ブロックチェーン上） |
| Bytecode（バイトコード） | スマートコントラクトの機械語命令列 |
| State Root（状態ルート） | 全アカウントの状態をまとめたハッシュ値 |

**State Root** はマークルツリー（木構造のデータ）の根（root）で、全てのアカウント残高・コードなどの要約です。

#### Layer 1 の長所と短所

| 評価 | 内容 |
|---|---|
| ✅ 安全（Secure） | 多数のノードが検証するため改ざん困難 |
| ✅ 分散（Decentralized） | 特定の管理者不在 |
| ❌ 高コスト（Expensive） | 全ノードが同じ処理をするため非効率 |
| ❌ 低速（Slow） | 合意形成に時間がかかる |

---

### 1.3 zk-Rollup（ゼロ知識ロールアップ）の発想

Layer 1の「高コスト・低速」問題を解決するのがzk-Rollupです。

**基本アイデア：**
- Layer 2（L2）で大量の取引を処理する
- その処理の「正しさ」を暗号学的証明（$\pi$）で保証する
- L1にはデータと証明だけを送る（全員が検証しなくてよい）

$$\text{L2で} \{TX_1, TX_2, \ldots, TX_n\} \text{を処理} \rightarrow \text{L1に (data, } \pi\text{) を送信}$$

しかし普通のzk-Rollupには問題があります：

- 通常のzk-Rollupのprover（証明者）は**独自の回路（circuit）**で作られる
- 複数のproverが互いに連携できない
- EVMと完全互換でない → 既存のDAppsが動かない

---

### 1.4 Scroll：ネイティブzkEVMソリューション

Scrollは**zkEVM**（ゼロ知識EVM）として、EVMをそのまま証明します。

**メリット：**
- **Developer friendly（開発者に優しい）**：既存ツール・言語がそのまま使える
- **Composability（組み合わせ可能性）**：DApps同士が連携できる

**デメリット（解決すべき課題）：**
- **Hard to build（構築が困難）**：EVMは証明向けに設計されていない
- **Large proving overhead（証明の計算コストが大きい）**

この2つのデメリットに対応するための技術が：
- 多項式コミットメント（Polynomial commitment）
- Lookup + カスタムゲート（Custom gate）
- ハードウェア加速（Hardware acceleration）
- 再帰的証明（Recursive proof）

---

### 1.5 zkEVM の種類（フレーバー）

Justin Drakeによる分類：

#### 言語レベル（Language level）
- Solidity/YulなどのEVM向け言語を**SNARK向け別VMにトランスパイル**
- EVMとは異なる命令セットで動く
- 例：Matter Labs（zkSync）、Starkware

#### バイトコードレベル（Bytecode level）― **Scrollはここ**
- EVMバイトコードを**直接解釈**する
- ただし一部のデータ構造をSNARK向けに置き換えるため、State Rootが若干異なる場合あり
- 例：**Scroll**、Hermez、Consensys

#### コンセンサスレベル（Consensus level）
- Ethereum L1のコンセンサスと**完全同等**を目指す
- L1のState Rootそのものの正当性を証明
- Ethereumの「zk-SNARKで全部やる」ロードマップの一部

---

## 2. zkEVM をゼロから構築する

### 2.1 ゼロ知識証明のワークフロー全体像

ゼロ知識証明（ZKP）では次の流れで証明を生成します：

$$\text{プログラム} \xrightarrow{\text{制約への変換}} \text{制約（Constraints）} \xrightarrow{\text{証明生成}} \text{証明} \pi$$

**制約の表現形式：**
- R1CS（Rank-1 Constraint System）
- **Plonkish**（Scrollが使用）
- AIR（Algebraic Intermediate Representation）

**証明システム：**
- Polynomial IOP（対話的証明プロトコル）+ PCS（多項式コミットメントスキーム）
- Scrollは **Plonk IOP + KZG コミットメント** を使用

---

### 2.2 R1CS（Rank-1 Constraint System）の基礎

R1CSとは、ゼロ知識証明の最も基本的な制約表現の一つです。

#### 証人（Witness）ベクトル

$$\mathbf{w} = (w_1, w_2, w_3, w_4, w_5, \ldots, w_{n-1}, w_n)$$

これは「私が知っている秘密の値のリスト」です。  
例えば、$w_1, w_2, w_3$ は入力、残りは中間計算値・出力です。

#### R1CSの制約式

$$(\underbrace{a_1 w_1 + \cdots + a_n w_n}_{A\mathbf{w}}) \cdot (\underbrace{b_1 w_1 + \cdots + b_n w_n}_{B\mathbf{w}}) = \underbrace{c_1 w_1 + \cdots + c_n w_n}_{C\mathbf{w}}$$

- 各制約は「線形結合 × 線形結合 ＝ 線形結合」の形
- 複数の制約を組み合わせてプログラムの全演算を表現

**具体例：**

$$
(2w_1 + 1) \cdot (3w_1 + 4w_2) = w_{n-2} + 2
$$
$$
(w_3 + 2) \cdot w_4 = w_n + 1
$$

**R1CSの意味（ZKP的に）：**
> 「私は $\{input, va, vb, vc, \ldots\}$ というベクトルを知っており、これは全ての制約を満たす」  
> ということを、ベクトルの中身を明かさずに証明できる

---

### 2.3 Plonkish 算術化（Arithmetization）

Plonkishは、R1CSよりも柔軟で表現力豊かな制約表現です。

#### 基本構造：表（テーブル）

```
| a₀      | a₁      | a₂      | a₃   | a₄     | T₀     | T₁     | T₂     | T₃     | T₄     |
|---------|---------|---------|------|--------|--------|--------|--------|--------|--------|
| input₀  | input₁  | input₂  |      | output |        |        |        |        |        |
| va₁     | vb₁     | vc₁     |      | vd₁    |        |        |        |        |        |
| va₂     | vb₂     | vc₂     |      | vd₂    |        |        |        |        |        |
| ...     | ...     | ...     |      | ...    |        |        |        |        |        |
```

- **左半分（オレンジ）**: witness（証人）= 秘密の計算値
- **右半分（水色）**: lookup table（ルックアップテーブル）= 事前に定義された正しい値の表

#### カスタムゲート（Custom gate）

複数の行・列にまたがる**高度な制約式**を表現できます。

**例1（3値の積）：**
$$va_3 \cdot vb_3 \cdot vc_3 - vb_4 = 0$$

**例2（積と和の組合せ）：**
$$vb_1 \cdot vc_1 + vc_2 - vc_3 = 0$$

**例3（異なる行にまたがる制約）：**
$$vc_1 + va_2 \cdot vb_4 - vc_4 = 0$$

カスタムゲートの特徴：
- **高次（High degree）**：3次以上の多項式制約が可能
- **高いカスタマイズ性（More customized）**：任意の演算を効率的に表現

#### パーミュテーション（Permutation）

「同じ値が別の場所にも存在する」ことを証明する仕組みです。

$$vb_4 = vc_6 = vb_6 = va_6$$

これにより、計算の途中で同じ値を何度も参照することを効率的に表現します。

#### ルックアップ引数（Lookup argument）

「この値は事前に定義した表の中に存在する」ことを証明します。

**例1：範囲チェック**

$$vc_7 \in [0, 15]$$

つまり「$vc_7$ は0から15の整数」という制約を、$T_0 = \{0000, 0001, 0010, \ldots, 1111\}$ というテーブルへのルックアップで表現。

**例2：XOR演算**

$$va_7 \oplus vb_7 = vc_7$$

XOR（排他的論理和）は通常の加減乗除では表現が難しいですが、全ての4ビットXOR組合せをテーブル $T_0 \times T_1 \times T_2$ として事前定義しておき、ルックアップで確認。

ルックアップ引数は以下にも使えます：
- 範囲チェック（range check）
- XOR・AND等のビット演算
- **RAM（ランダムアクセスメモリ）操作**

---

### 2.4 EVMを証明するために必要なこと

#### 「フロントエンド」選択の課題

EVMには証明を難しくする特性があります：

| EVMの特性 | 証明上の課題 | 解決アプローチ |
|---|---|---|
| ワードサイズが256ビット | 効率的な範囲証明が必要 | リムへの分割・ルックアップ |
| zkに不向きなopcode（SHA3等） | 回路間の効率的な接続 | Lookup table + 別回路 |
| 読み書きの整合性（Stack/Memory/Storage） | 効率的なマッピング | RAM回路 |
| 実行トレースが動的 | 効率的なON/OFFスイッチ | カスタムゲートのセレクタ |

#### 証明すべきこと：状態遷移

$$\text{World State}(t) \xrightarrow{\text{Transaction}} \text{Computation（EVM）} \rightarrow \text{World State}(t+1)$$

具体的には：
1. 世界状態 $t$ のState Root（$Root$）
2. トランザクション情報（from, to, gas, value等）
3. EVM実行により生成された**実行トレース**（step-by-step命令列）
4. 世界状態 $t+1$ の新しいState Root（$Root'$）

**実行トレースの例：**

| Step | Opcode    |
|------|-----------|
| 0    | PUSH1 80  |
| 1    | PUSH1 40  |
| 2    | MSTORE    |
| 3    | CALLVALUE |
| ...  | ...       |
| n    | RETURN    |

zkEVMは「このトレースが正しく、正しい状態遷移を行った」ことを証明します。

---

### 2.5 EVM回路の設計

各実行ステップ（Step 1, Step 2, ..., Step n）は回路の行に対応します。

#### 各ステップの構造

```
┌─────────────────────────────────────────────────┐
│ Step context（ステップのコンテキスト）            │
│   Codehash | gas | PC | SP | root               │
├─────────────────────────────────────────────────┤
│ Case switch（命令選択スイッチ）                   │
│   sADD | sSUB | sMUL | ... | sErr1 | sErr2 | ..│
├─────────────────────────────────────────────────┤
│ Opcode specific witness（命令固有の証人）         │
│   a_lo | a_hi | b_lo | b_hi | c_lo | c_hi | ..│
└─────────────────────────────────────────────────┘
```

#### ① Step context（ステップコンテキスト）

各ステップで必ず存在する情報：

- **Codehash**: 実行中のスマートコントラクトのハッシュ
- **Gas**: 残りガス量（EVMの実行コスト）
- **PC（Program Counter）**: 次に実行する命令の位置
- **SP（Stack Pointer）**: スタックの現在位置

#### ② Case switch（命令ケーススイッチ）

「どの命令を実行しているか」を示すスイッチ。  
**必ず1つだけON**になる（ブール制約）：

$$sADD \cdot (1 - sADD) = 0$$
$$sMUL \cdot (1 - sMUL) = 0$$
$$\vdots$$
$$sADD + sMUL + \cdots + sERRk = 1$$

- `sADD = 1` ならADD命令、`sMUL = 1` ならMUL命令、という形
- エラーケース（`sErr1, sErr2, ...`）も含む

#### ③ Opcode specific witness（命令固有の証人）

各命令に必要な追加データ。  
ADDの場合：

$$a_{lo}, a_{hi}, b_{lo}, b_{hi}, c_{lo}, c_{hi}$$

（$a, b$ がオペランド、$c$ が結果、`lo/hi` は256ビットを128ビット×2に分割したもの）

---

### 2.6 ADD命令の回路制約（詳細例）

256ビット整数の加算 $a + b = c$ を証明するケース：

#### Step contextの制約

```
sADD * (pc' - pc - 1) == 0      // PCが1増える
sADD * (sp' - sp - 1) == 0      // スタックポインタが1減る（2個取り出し1個追加）
sADD * (gas' - gas - 3) == 0    // ガスを3消費
```

（`sADD *` がかかっているのは、ADD命令の時だけこの制約を有効にするため）

#### Opcode specific witness の制約

256ビット整数を128ビット2つ（lo/hi）に分割して計算：

$$sADD \cdot (a_{lo} + b_{lo} - c_{lo} - carry_0 \cdot 2^{128}) = 0$$

$$sADD \cdot (a_{hi} + b_{hi} + carry_0 - c_{hi} - carry_1 \cdot 2^{128}) = 0$$

これは筆算の繰り上がり（carry）付き加算の回路表現です。

#### RAM（メモリ）との接続

ADD命令はスタックから2値を読み取り（Read）、結果をスタックに書く（Write）：

| idx | tag   | addr | R/W | value  |
|-----|-------|------|-----|--------|
| 5   | STACK | 1022 | 0   | word_a |
| 6   | STACK | 1023 | 0   | word_b |
| 7   | STACK | 1023 | 1   | word_c |

この読み書きの整合性をRAM回路が保証します。

---

### 2.7 SHA3（Keccak）のような特殊命令

SHA3はビット演算が多く、通常の算術回路では非常に非効率です。

**解決策：**

1. EVM回路は「入力と出力のハッシュ値のペア」だけを保持
2. Keccak専用の**別回路（Hash circuit）**が実際の計算を担当
3. EVM回路はその結果を**Hash lookup table**で参照する

```
EVM回路の opcode specific witness:
  input | Hash(input)
       ↓（ルックアップ）
  Hash lookup table ←── Hash circuit（が計算して書き込む）
```

---

### 2.8 zkEVM 回路の全体アーキテクチャ

```
                    EVM circuit（中心）
                         |
          ┌──────────────┼──────────────────┬──────────────┐
          ↓              ↓                  ↓              ↓
    Fixed table    Keccak table        RAM table      Bytecode table
   （ビット演算・    （SHA3の入出力）  （stack/memory/  （pc, opcode）
    範囲チェック）                      storage）
                         ↑                  ↑
                   Keccak circuit      RAM circuit
                   MPT circuit
          
    Transaction table ←── Transaction circuit ←── ECDSA circuit
    （署名情報）                                   （Signature table）
    
    Block context table ←── Block circuit
    （ブロック情報）
```

- **EVM circuit**: 中心的な回路。全てのopcodeを処理
- **RAM circuit**: Stack・Memory・Storageの読み書き整合性を保証
- **Keccak circuit**: Keccak（SHA3）ハッシュを計算
- **MPT circuit**: Merkle Patricia Tree（状態木）を証明
- **Bytecode circuit**: バイトコードの正当性を保証
- **Transaction circuit**: トランザクションの形式検証
- **ECDSA circuit**: デジタル署名の検証（secp256k1曲線）
- **Block circuit**: ブロックコンテキストの検証

---

### 2.9 2層アーキテクチャの証明システム

zkEVMの証明は2段階で生成されます：

```
┌──────────────────────────────────┐
│  zkEVM（第1層：回路証明）         │
│                                  │
│  EVM Circuit  ─────┐             │
│  RAM Circuit  ─────┼──→ Aggregation Circuit（第2層）
│  Storage Cir. ─────┘             │
│  Other Circuits ───┘             │
└──────────────────────────────────┘
```

#### 第1層（Halo2-KZG、Poseidonハッシュ）の特性

| 要件 | 内容 |
|---|---|
| 表現力が高い | カスタムゲート・ルックアップをサポート |
| 高速証明 | GPU proverで高速化 |
| 検証回路が小さい | 次の層での証明が容易 |
| ユニバーサルトラステッドセットアップ | 一度セットアップすれば使い回せる |

#### 第2層（Halo2-KZG、Keccakハッシュ）の特性

| 要件 | 内容 |
|---|---|
| 検証効率が高い | EVMのスマートコントラクトで検証可能 |
| 証明サイズが小さい | ガスコストが低い |
| 非ネイティブ演算を効率的に表現 | 第1層の検証を第2層で効率的に証明 |

#### 実際の規模

**第1層（EVM circuit）：**
- 列数：**116列**
- カスタムゲート数：**2,496個**
- ルックアップ数：**50個**
- 最高ゲート次数：9
- 1Mガスで：**$2^{18}$行**（ガスが増えるほど行が増える）

**第2層（Aggregation circuit）：**
- 列数：**23列**
- カスタムゲート数：**1個**
- ルックアップ数：**7個**
- 最高ゲート次数：5
- 第1層証明の集約に：**$2^{25}$行**

#### GPU証明の性能

| 対象 | CPU証明時間 | GPU証明時間 | 高速化率 |
|---|---|---|---|
| EVM circuit | 270.5秒 | **30秒** | **9倍** |
| Aggregation circuit | 2,265秒 | **149秒** | **15倍** |

**1Mガスのブロック全体：** 第1層2分 + 第2層3分 = 約5分で証明生成

GPU最適化の内容：
- MSM（Multi-Scalar Multiplication）カーネルの高速化
- NTT（Number Theoretic Transform）カーネルの高速化
- CPU-GPU計算のパイプライン化
- マルチGPUカード対応・メモリ最適化

---

## 3. 面白い研究課題

### 3.1 ランダム性の取り扱い（RLC）

#### 256ビット整数の表現問題

EVMのワードサイズは256ビットですが、証明に使う有限体の要素は通常それより小さい。  
そのため256ビット整数を **32個の8ビットリム（limb）** に分割します：

$$A = a_0 + a_1 \cdot 256 + a_2 \cdot 256^2 + \cdots + a_{31} \cdot 256^{31}$$

#### RLC（Random Linear Combination）

リムを1つの値に圧縮するために、ランダムな値 $\theta$ を使います：

$$A_{RLC} \equiv a_0 + a_1 \cdot \theta + a_2 \cdot \theta^2 + \cdots + a_{31} \cdot \theta^{31} \pmod{F_p}$$

**重要：** $\theta$ は $a_0, \ldots, a_{31}$ が全て確定した後に計算される必要があります。

これを実現するために**マルチフェーズ証明者（Multi-phase prover）** を使います：
1. フェーズ1：$a_0, \ldots, a_{31}$ を確定させる
2. $\theta$ を導出する（コミットメントから）
3. フェーズ2：$A_{RLC}$ を計算する

**RLCの用途：**
- 256ビットワードを1つの値に圧縮
- 動的長さの入力をエンコード（Keccakへの入力など）
- ルックアップテーブルのレイアウト最適化

**今後の課題：** RLCを除去してより単純な方式（hi/loの2値表現など）に置き換えられるか？

---

### 3.2 回路レイアウトの最適化

#### 動的実行トレースの問題

EVMの実行トレースは動的です：
- 命令によってステップサイズが異なる
- パーミュテーション（コピー制約）が固定でない
- 標準的なゲートが使いにくい

現在のScrollの課題：
- **2,000以上のカスタムゲート**を管理
- 異なる行へのrotation（参照）でセルにアクセス

より良いレイアウト方法の研究が必要です。

---

### 3.3 動的サイズの問題

zkEVMの回路サイズは事前に固定する必要があります（Plonkの制約）。

**問題：**
- Keccakハッシュの最大回数が固定 → 少ない場合はパディングが無駄
- MLOADのような命令はコストが高い（行数が多い）
- パディングのための余分な証明コストがかかる

**研究課題：zkEVMを動的サイズに対応できるか？**

---

### 3.4 証明者のハードウェア・アルゴリズム

#### 現状の問題

- GPU で MSM・NTT を高速化すると、ボトルネックが**証人生成**と**データコピー**に移る
- 必要CPU メモリ：1TB → 最適化で300GB以上

#### 理想的な証明者の要件

- **並列化可能（Parallelizable）**：分散処理に対応
- **低ピークメモリ（Low peak memory）**：安価なマシンで動く
- **証人生成を軽視しない**：計算全体を最適化
- **安価なマシンで動作**：より分散化が可能

#### 今後の研究方向

1. **小さい有限体への移行？**
   - Goldilocks素体（$p = 2^{64} - 2^{32} + 1$）
   - Mersenne素体（$p = 2^{31} - 1$）
   - Breakdown/FRI と組み合わせ

2. **楕円曲線ベースの構成を維持？**
   - SuperNova
   - 高速MSMを持つサイクリック楕円曲線

3. **異なる証明システムの組合せ方**
   - 第1層：表現力優先
   - 第2層：検証効率優先

---

### 3.5 セキュリティ：監査と健全性

#### コードリスクの現実

Vitalik（イーサリアム創始者）も指摘：

> **「34,469行のコードが長期間バグゼロでいることはない」**

PSEのZK-EVM circuits（参考実装）は34,469行にもなります。

#### 最善の監査方法の研究課題

- **手動監査（Audit Manually）**：人間が回路を読む
- **形式検証（Formal verification）**：数学的に証明する

特に「VMの命令（IR）に基づく回路全般」の最善監査方法はまだ確立されていません。

---

## 4. zkEVM のその他の応用

### 4.1 zkRollup（スケーリング）

最も直接的な応用：

$$\text{L2でn個のTxを実行} \rightarrow \text{zkEVMで証明} \rightarrow \text{L1のスマートコントラクトで検証}$$

- L1にはデータと証明だけを送信
- L1コストを大幅削減（多数のTxを一括証明）

---

### 4.2 ブロックチェーンの「神聖化（Enshrine）」

#### 発想

L1のブロックそのものをzkEVMで直接証明する：

$$\pi_{\text{block}[t]} \text{を内包したブロック} \rightarrow \pi_{\text{block}[t+1]} \text{を内包したブロック} \rightarrow \cdots$$

#### 再帰的証明

「前のブロックの証明が正しい」という事実も含めて証明することで、**ブロックチェーン全体を1つの証明にまとめられる**。

$$\pi_{\text{chain}} = \text{Verify}(\pi_{\text{block}[n]}, \text{Verify}(\pi_{\text{block}[n-1]}, \ldots))$$

これはEthereumの「zk-SNARKで全てを証明する」長期ロードマップの一部です。

---

### 4.3 エクスプロイトの証明（Proof of Exploit）

```
World State(t) + Transaction → World State(t+1)
```

**応用：**

「私はあるトランザクションを知っており、それを実行するとState Rootが $Root'$ に変わる（例：あなたの残高が0になる）」ということを、**実際にトランザクションを実行・公開せずに証明できる**。

用途：
- セキュリティ研究者がバグを証明しながら悪用を防ぐ
- Bug bounty（バグ報奨金）の提示
- ホワイトハットハッカーの活動

---

### 4.4 attestation（「zk oracle」）

#### 概念

ブロックチェーンの過去の状態データを**信頼できる形で読み取る**ための仕組み。

```
Layer 1の過去のブロック
       ↓（State Root を信頼する）
ZK回路が過去のデータを読み取り・計算・検証
       ↓
L1スマートコントラクトで証明を検証
```

**例：Axiom（zk co-processor）**
- L1の過去の残高・取引履歴などを読み取る
- 複雑な集計計算をoff-chainで実行
- 結果の正当性をon-chainで検証

これにより「過去n日間の平均取引量が閾値以上」などの複雑な条件をスマートコントラクトで使えるようになります。

---

## まとめ：Scrollの全体像

| レイヤー | 技術 | 役割 |
|---|---|---|
| EVM実行 | EVM互換の実行環境 | Layer 2でのトランザクション処理 |
| 回路設計 | Plonkish算術化 | EVMの実行を制約式に変換 |
| 証明（第1層） | Halo2-KZG + Poseidon | EVM/RAM/Storage等の各回路証明 |
| 集約（第2層） | Halo2-KZG + Keccak | 全証明を1つに集約 |
| 高速化 | GPU prover | MSM・NTTの並列処理 |
| L1検証 | Solidity smart contract | 集約証明の最終検証 |

**Scrollの現状（講義時点）：**
- テストネットで稼働中
- 本番レベルのインフラ整備済み
- GPU proverで5分以内に1Mガスブロックの証明生成可能

---

## 補足：用語集

| 用語 | 意味 |
|---|---|
| ZKP（Zero Knowledge Proof） | ゼロ知識証明：知識を明かさずに知っていることを証明 |
| zkSNARK | 簡潔な非対話的ゼロ知識証明 |
| EVM | Ethereum Virtual Machine：イーサリアムの仮想コンピュータ |
| zkEVM | EVM実行を証明するZKP回路 |
| R1CS | Rank-1 Constraint System：制約表現の一形式 |
| Plonkish | 柔軟な制約表現。カスタムゲートとルックアップをサポート |
| KZG | Kate-Zaverucha-Goldberg多項式コミットメント |
| MSM | Multi-Scalar Multiplication：楕円曲線演算の最重要操作 |
| NTT | Number Theoretic Transform：高速有限体多項式乗算 |
| MPT | Merkle Patricia Tree：イーサリアムの状態を格納する木構造 |
| ECDSA | 楕円曲線デジタル署名アルゴリズム |
| RLC | Random Linear Combination：複数値を1つに圧縮する技法 |
| L1/L2 | Layer 1（基盤ブロックチェーン）/ Layer 2（上位スケーリング層） |
| Rollup | 多数のL2取引をまとめてL1に記録する方式 |
| State Root | 全アカウント状態のマークル根（要約ハッシュ） |
| Opcode | EVMの基本命令（ADD, MUL, PUSH, MSTORE等） |
| Gas | EVMの計算コストの単位 |
| Prover | 証明を生成する計算機 |
| Verifier | 証明を検証する（通常はスマートコントラクト） |
