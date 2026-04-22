---
title: "[ゼロ知識証明入門シリーズ 27/30] Zcash — 完全秘匿トランザクションの仕組み"
emoji: "🪙"
type: "tech"
topics:
  - zkp
  - zcash
  - privacy
  - snark
  - nullifier
published: false
---

**日付**: 2026年4月22日
**学習内容**: **Zcash** は 2016 年に始まった、ZKP を使った**世界初の大規模プライバシー暗号通貨**。送信者・受信者・送金額のすべてを ZK-SNARK で隠しながら、「正当な送金である」ことだけを公開検証できる。本記事では **(1) 透明ブロックチェーンのプライバシー問題**、**(2) コインの shielded 化（coin commitment）**、**(3) Nullifier による二重支払い防止**、**(4) Sprout / Sapling / Orchard の 3 世代進化**、**(5) Sapling の Spend Description と Output Description**、**(6) Orchard の Halo2 ベース設計**、**(7) Zcash のガバナンスと現状**、**(8) プライバシーと規制の緊張**を扱う。

## 0. 本記事の位置づけ

ZKP 応用の最初の成功例が Zcash (Zerocash プロトコル)。

- 2014 年: Zerocash 論文（Ben-Sasson et al.）
- 2016 年: Zcash ローンチ
- 2018 年: Sapling アップグレード（プライバシーとパフォーマンス向上）
- 2022 年: Orchard アップグレード（Halo2 ベース）

構成:

- **第1章**: 透明コインの問題
- **第2章**: Coin Commitment の仕組み
- **第3章**: Nullifier
- **第4章**: 送金トランザクション
- **第5章**: Sprout 世代
- **第6章**: Sapling 世代
- **第7章**: Orchard 世代
- **第8章**: 規制と現状
- **第9章**: Q&A とまとめ

## 1. 透明コインの問題

### 1.1 Bitcoin のプライバシー

Bitcoin の全トランザクションは**公開**:

- 誰が誰にいくら送ったかが永久に記録
- アドレスは擬匿名（pseudonymous）だが、**解析で実名と紐付けられる**
- **受け取った給料・家賃・医療費**などがすべて明らか

### 1.2 解析の具体例

- Chainalysis, Elliptic などが大口アドレスをラベリング
- **Taint analysis**: ダークマーケット由来の資金を追跡
- KYC 済み取引所で入出金すれば実名と繋がる

### 1.3 Mixer の限界

Tornado Cash のような mixer で混ぜても:

- タイミング分析
- 金額の整合性
- 出金パターンで特定

本質的にプライベートな**暗号通貨プロトコル**が必要 → Zcash。

## 2. Coin Commitment の仕組み

### 2.1 Shielded Pool

Zcash には **transparent pool** と **shielded pool** がある:

- **Transparent**: Bitcoin と同じ、全公開
- **Shielded**: ZKP でプライバシー保護

### 2.2 Coin Commitment (Note)

Shielded プールでは、コインは**commitment** $\text{cm}$ として表現:

$$
\text{cm} = \text{Commit}(\text{value}, \text{receiver\_pk}, \rho, r)
$$

- `value`: 金額
- `receiver_pk`: 受信者の公開鍵
- $\rho$: 一意性のための乱数（Nullifier の元）
- $r$: commitment 乱数（hiding）

### 2.3 Commitment の性質

- **Hiding**: $\text{cm}$ から値が漏れない
- **Binding**: あとで値を変えられない

Pedersen commitment や Poseidon コミットメントを使用。

### 2.4 Commitment Tree

すべての commitment は**Merkle tree** に集約される:

```
                root
               /    \
            h1        h2
           /  \      /  \
        cm1  cm2  cm3  cm4 ...
```

Merkle root はチェーン上で公開。各 commitment は root から位置証明で参照可能。

## 3. Nullifier

### 3.1 問題: 二重支払い防止

Shielded プールでも「同じ coin を 2 回使う」ことを防ぐ必要。しかし「どの coin か」は秘密なので、**普通のチェックはできない**。

### 3.2 Nullifier の仕組み

各 coin には**一意の nullifier** を対応:

$$
\text{nf} = \text{PRF}_{\text{sk}}(\rho)
$$

- PRF: 疑似乱数関数（Poseidon ベースなど）
- $\text{sk}$: 所有者の秘密鍵
- $\rho$: commitment 内の乱数

### 3.3 Spend 時の処理

所有者が送金（spend）するとき:

1. ZKP で「この nullifier は自分の coin の正当な nullifier」を証明
2. nullifier をチェーンの**nullifier set**に追加
3. 以後、同じ nullifier は拒否される

### 3.4 プライバシーの保証

- Nullifier は ZKP 内部で計算
- 外部からは「どの coin」か分からない
- **Unlinkability**: 送金元と送金先がリンクされない

### 3.5 Double-spending 防止

同じ coin は同じ nullifier を生成。2 回使おうとすると nullifier set が衝突 → 拒否。

```mermaid
flowchart LR
    Coin[Coin<br/>cm, ρ]
    User[User<br/>sk]
    NF[Nullifier<br/>nf = PRF_sk(ρ)]
    Chain[Chain]
    
    Coin --> NF
    User --> NF
    NF --> Chain
    Chain -->|Already in set?| Reject[Reject]
    Chain -->|Not yet| Accept[Accept + Add]
```

## 4. 送金トランザクション

### 4.1 Tx の構成

Shielded transaction は:

- **Inputs**: 消費する coin の nullifier
- **Outputs**: 新しい coin の commitment
- **ZK Proof**: 全体の正当性証明
- **Value balance**: 入出力の合計が一致

### 4.2 ZKP が証明すること

1. 入力の coin commitment が**過去の commitment tree にある**（Merkle path）
2. 入力の nullifier は正しく計算されている
3. 出力の commitment は正しく計算されている
4. **入力の合計 = 出力の合計 + 手数料**（value conservation）
5. 送信者が入力 coin の**所有者**（秘密鍵を知る）

### 4.3 何が漏れない

- 送信者のアドレス
- 受信者のアドレス
- 金額
- 使用した coin

### 4.4 何が漏れる

- トランザクションの存在
- チェーン上でのタイミング
- 手数料（透明）
- 公開入力（Merkle root, nullifier, ZKP）

## 5. Sprout 世代 (2016-2018)

### 5.1 技術スタック

- **SNARK**: BCTV14（Parno et al. 2013 の改良版）
- **曲線**: BN254
- **ハッシュ**: SHA-256

### 5.2 制限

- **Trusted Setup**: 回路固有（後述の ceremony で実施）
- **送金の重さ**: 1 tx 生成に数分
- **メモリ**: 数 GB 必要
- **ZK 性のない transparent-to-shielded** は可

### 5.3 "Powers of Tau" Ceremony

Zcash Sprout launch 時に、**6 人の Zcash チーム**が分散で CRS を生成:

1. 各参加者がランダム $\tau_i$ を選ぶ
2. 順番に掛け算（$\tau = \tau_1 \cdot \tau_2 \cdot \ldots$）
3. 最終 CRS が完成

## 6. Sapling 世代 (2018-2022)

### 6.1 大幅な改善

Sapling は Sprout の問題を解決:

| 項目 | Sprout | Sapling |
|---|---|---|
| SNARK | BCTV14 | **Groth16** |
| 曲線 | BN254 | **BLS12-381** |
| ハッシュ | SHA-256 | **Pedersen / BLAKE2s** |
| 送金時間 | 数分 | **数秒** |
| メモリ | 数 GB | **40 MB** |
| ZK モバイル | 困難 | **可能** |

### 6.2 Spend Description

送金の**入力側**（spend）の証明データ:

- Nullifier $\text{nf}$
- 電子署名の公開鍵 `rk`
- Merkle root $\text{anchor}$
- Zero-knowledge proof $\pi_{\text{spend}}$

### 6.3 Output Description

送金の**出力側**（output）の証明データ:

- Commitment $\text{cm}$
- エフェメラル公開鍵 `epk`
- Encrypted note（受信者がデコード）
- Zero-knowledge proof $\pi_{\text{output}}$

### 6.4 RedJubjub / Jubjub

Sapling は新しい埋め込み楕円曲線 **Jubjub** を使う。BLS12-381 の上で軽い楕円曲線演算ができる。**RedJubjub** 署名で spend 権限を証明。

### 6.5 Sapling Ceremony

2018 年、**90 人以上の参加者**が加わった分散セレモニー。1 人でも正直なら安全。

## 7. Orchard 世代 (2022-)

### 7.1 Halo2 ベース

Orchard は **Halo2**（ECC の新 SNARK ライブラリ）をベースに:

- **Plonkish arithmetization**
- **IPA (Inner Product Argument)** ベース PCS
- **Recursive composition** で再帰可能
- **No Trusted Setup** (Transparent)

### 7.2 Pallas / Vesta 曲線

**Pasta** 曲線ペア（Pallas + Vesta）を使用。Article 7, 19 で触れた再帰 SNARK の基盤。

### 7.3 Improvements

- **No Trusted Setup** → ceremony 不要
- **Recursive proofs** → scalability 向上
- **Unified sendable receivers** → アドレス管理が楽
- **ZIP 224**: 改善された action description

### 7.4 Orchard Action

Orchard の各アクション:

- Spend 入力（Sapling の spend に相当）
- Output 出力（Sapling の output に相当）
- **1 つの unified action** で両方を 1 つの ZK proof に包む

### 7.5 性能

- 送金時間: 数秒
- Proof サイズ: ~3.5 KB (Halo2 の特性)
- モバイルでも生成可能

## 8. 規制と現状

### 8.1 規制圧力

- **OFAC 制裁**: Tornado Cash が 2022 年に制裁対象
- **FATF ガイドライン**: プライバシーコインに対する警戒
- **取引所の対応**: Binance, Kraken などで Zcash の shielded 機能を制限

### 8.2 View Keys

Zcash は**selective disclosure** をサポート:

- **Incoming Viewing Key**: 自分宛 tx だけ閲覧可能
- **Full Viewing Key**: 全 tx 閲覧可能

税務申告や監査のため、**一部だけ開示**できる。

### 8.3 Shielded Asset (ZSA)

最近の Zcash で、**カスタムトークン**を shielded pool で扱える拡張。Zcash 上の stablecoin など。

### 8.4 Mobile Wallets

- **Zingo** (iOS/Android)
- **Nighthawk**
- **ZecWallet**

Orchard になってからモバイルで shielded tx が快適。

### 8.5 統計

- Zcash の shielded pool 使用率は歴史的に低かった（数%〜20%）
- Orchard 以降、shielded use が増加傾向
- 2024 年: shielded-only モードへの段階的移行計画

## 9. Q&A

### Q1: Zcash は本当に追跡不可能？

**実装上 YES**。ZKP が壊れない限り、チェーン解析では誰が送ったか分からない。ただし:

- Transparent と shielded を混ぜると漏れる
- IP アドレス・タイミング解析で推定される可能性
- 量子計算機が来ると Groth16 が破綻（Orchard は Halo2 なので相対的に強い）

### Q2: なぜ Zcash は Monero より目立たない？

- **Shielded pool 使用が任意**: 多くの tx は transparent で、Monero のような強制プライバシーとは違う
- **中央集権的な運営**: Electric Coin Company (ECC) が主導
- **規制対応**: View key の存在で CBDC 等にも親和的

Monero は Ring Signature ベースで全 tx がプライベート。

### Q3: Trusted Setup のリスクは？

- Sprout/Sapling の ceremony で**1 人でも毒 $\tau$ を保持している**と、偽 coin を mint できる
- Orchard は Transparent なのでこの問題なし

実用上、Zcash の ceremony は分散度が高く、信頼できる。

### Q4: Zcash と Ethereum Privacy の比較？

- Zcash: プロトコル自体が shielded
- Ethereum の Tornado Cash: アプリレイヤで shielded
- Aztec: ZKP でプライベートスマートコントラクト

Zcash は coin 送金のみ、Ethereum 系はより複雑なロジック可能。

### Q5: Halo2 と Orchard の関係は？

Halo2 は Zcash 財団が開発した汎用 SNARK ライブラリ。Zcash Orchard はその最初の実用例。後に Scroll, Taiko など Ethereum L2 でも採用。

### Q6: Zcash の学ぶ価値は？

**ZKP 応用の古典**。歴史的経緯（BCTV → Groth16 → Halo2）を追うと SNARK の進化が分かる。ソースコードは公開で、実装の参考になる。

## 10. まとめ

### 本記事の要点

1. **Zcash** = ZKP を使ったプライバシー暗号通貨の先駆け
2. **Coin Commitment** でコインを値・受信者・乱数の hash で表現
3. **Nullifier** で二重支払い防止、ZKP で一意性を証明
4. **Sprout (2016)**: BCTV14 + BN254、重い
5. **Sapling (2018)**: Groth16 + BLS12-381、大幅高速化
6. **Orchard (2022)**: Halo2 + Pasta、Transparent Setup + 再帰
7. **View Keys** で選択的開示、規制対応
8. ZKP 応用の歴史的クラシック

### 次の記事（Article 28）へ

次の記事は **Tornado Cash** と **Mixer プロトコル** の詳細。Zcash と違って Ethereum L1 上のコントラクトで動き、より汎用的な shielded pool として機能する。

### 3行サマリ

- **Zcash = ZKP で送信者・受信者・金額すべてを隠す暗号通貨**
- **Coin Commitment + Nullifier + ZK-SNARK** が基本構造
- **3 世代**: Sprout → Sapling → Orchard、SNARK 進化の歴史を体現

---

## 参考文献

- Eli Ben-Sasson et al. *Zerocash: Decentralized Anonymous Payments from Bitcoin.* IEEE S&P 2014.
- Electric Coin Company. *Zcash Protocol Specification.* 2024.
- Zcash Foundation. *Orchard Action Description.* 2022.
- Electric Coin Company. *The Halo2 Book.*
- ZKP MOOC Lecture 2 (UC Berkeley, 2023).
