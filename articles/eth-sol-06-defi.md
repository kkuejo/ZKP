---
title: "[Ethereum vs Solana 徹底比較 6/8] DeFiエコシステム — Uniswap vs Raydium、AMMとCLOB"
emoji: "💰"
type: "tech"
topics:
  - defi
  - uniswap
  - raydium
  - amm
  - clob
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事は両チェーンの **DeFi エコシステム** を比較する。Ethereum は **Uniswap を起点とする AMM (Automated Market Maker) 文化** を全 EVM チェーンに展開した。Solana は **Serum / OpenBook / Phoenix の CLOB (Central Limit Order Book) 文化** と **Raydium のハイブリッド AMM** を育ててきた。両者は **DEX 設計・MEV 環境・TVL 動向・手数料構造** すべてで異なる。本記事では **(1) Ethereum DeFi の全体像**、**(2) Solana DeFi の全体像**、**(3) AMM vs CLOB**、**(4) 具体的比較 (Uniswap vs Raydium vs Phoenix)**、**(5) TVL 推移とエコシステム**、**(6) MEV 環境の差**、**(7) ステーキングと利回り**、**(8) 相互運用性 (bridge)** を扱う。

## 0. 本記事の位置づけ

Part 1-5 で両チェーンの **技術的違い** を見てきた。本記事では **実際にお金が動く場所 = DeFi** がどう違うかを見る。

DeFi の設計思想は、下層のチェーンの性質で決まる:
- **Ethereum**: 高ガス代 → **AMM (少ない tx で自動取引)**
- **Solana**: 低ガス代 + 高速 → **CLOB (CEX と同じ注文板)**

この必然的な違いが、両エコシステムの性格を決めている。

構成:

- **第1章**: Ethereum DeFi の全体像
- **第2章**: Solana DeFi の全体像
- **第3章**: AMM vs CLOB
- **第4章**: Uniswap vs Raydium vs Phoenix
- **第5章**: TVL と市場シェア
- **第6章**: MEV
- **第7章**: ステーキング
- **第8章**: Bridge
- **第9章**: Q&A とまとめ

## 1. Ethereum DeFi の全体像

### 1.1 王者 Uniswap

**Uniswap** (2018~) は DeFi の祖。**AMM** モデルを発明・普及:

- v1 (2018): ETH-ERC20 ペアのみ
- v2 (2020): 任意の ERC20 ペア、DEX の標準に
- v3 (2021): **Concentrated Liquidity** (価格帯指定 LP)
- v4 (2024): **Hooks** でカスタム挙動

### 1.2 主要 DeFi レゴ

Ethereum の DeFi は「レゴブロック」と呼ばれる:

| カテゴリ | 代表プロトコル |
|---|---|
| DEX | Uniswap、Curve、Balancer、Sushiswap |
| Lending | Aave、Compound、Morpho |
| Stablecoin | DAI (MakerDAO)、USDC、USDT、crvUSD |
| Derivatives | GMX、dYdX v3、Synthetix |
| Yield | Yearn、Convex、Pendle |
| Liquid Staking | Lido、Rocket Pool、Frax |
| Restaking | EigenLayer、Symbiotic |

相互に組み合わさって使われる。

### 1.3 EVM チェーンへの水平展開

Ethereum の DeFi は **他 EVM チェーンにコピー** された:

- **Polygon**: QuickSwap (Uniswap v2 fork), Aave
- **BNB Chain**: PancakeSwap (Uniswap v2 fork)
- **Arbitrum**: Uniswap, Aave, GMX ネイティブ
- **Optimism**: 同上
- **Base**: Aerodrome (Solidly fork)、Uniswap
- **Avalanche**: Trader Joe、Benqi

**同じコードが動く** ため、流動性が分散しつつも連携しやすい。

### 1.4 TVL

2026 年時点（概算）:
- **Ethereum L1**: $50-80B
- **Ethereum L2s 合計**: $30-50B
- **Ethereum エコシステム全体**: **$100B+**

世界の DeFi TVL の **50-60%** を占める（2024 年後半に一時 Solana の追い上げあり）。

### 1.5 成熟したインフラ

- **セキュリティ**: 数十億ドルをハックに奪われた教訓の蓄積
- **監査会社**: Trail of Bits、OpenZeppelin、Consensys Diligence など
- **保険**: Nexus Mutual、Risk Harbor
- **自動化**: Gelato、Keep3r

## 2. Solana DeFi の全体像

### 2.1 主要プロトコル

| カテゴリ | 代表プロトコル |
|---|---|
| DEX (AMM) | **Raydium**、**Orca**、Meteora |
| DEX (CLOB) | **OpenBook** (Serum の後継)、**Phoenix** |
| DEX Aggregator | **Jupiter** (超重要) |
| Lending | **Kamino**、MarginFi、Solend |
| Stablecoin | USDC (native)、USDT、**PYUSD**、**JitoSOL** (LST) |
| Derivatives | **Drift**、Zeta |
| Liquid Staking | **Jito**、Marinade、BlazeStake |
| Perp DEX | Drift、Jupiter Perps、Flash Trade |

### 2.2 DEX aggregator 文化

**Jupiter** は Solana DeFi の要:
- ユーザーは Jupiter 経由で swap
- Jupiter が **全 DEX を横断** して最良価格を提示
- **Raydium、Orca、Phoenix、OpenBook、Meteora** をすべて参照

= ユーザーは 1 つの UI で、内部的には全 DEX の流動性を使う。

### 2.3 ミームコイン経済

2024 年に爆発した **pump.fun** 現象:
- 誰でも数秒でミームコインを作れる
- **1 日あたり数万コインが発行**
- 2024 Q3 の Solana DEX volume 急増の主因

ミームコインは Ethereum では gas が高すぎて成立しない。**Solana の低コスト** が新しい経済圏を生んだ。

### 2.4 TVL

2026 年時点（概算）:
- **Solana**: **$8-15B**
- ピーク: 2024-12 に $12B 超、月間 DEX volume で **Ethereum を抜いた**

絶対値では Ethereum にまだ及ばないが、**volume ベースでは互角**。

### 2.5 成熟度

Ethereum と比べると:
- **プロトコル数**: 約 1/3
- **監査会社**: OtterSec、Zellic、Halborn、Neodyme など急成長
- **保険**: まだ限定的

## 3. AMM vs CLOB

### 3.1 AMM (Automated Market Maker)

**AMM**:
- **プール** に資産を入れる (例: ETH + USDC)
- 数式 (constant product `x*y=k`) で価格決定
- スワップはプールと取引（相手探し不要）

**利点**:
- シンプル、常に流動性あり
- **Lazy LP**: 放置で fee が入る
- オンチェーンで完結（tx 数少ない）

**欠点**:
- **Impermanent Loss** (価格変動で LP が損する)
- **大口取引でスリッページ**
- **価格発見が遅い** (アービトラージに依存)

### 3.2 CLOB (Central Limit Order Book)

**CLOB**:
- ユーザが **limit order** を置く
- **板 (order book)** に並ぶ
- 対向注文が来るとマッチング

**利点**:
- CEX と同じ UX
- **スリッページ少**
- **価格発見が速い**
- **メーカー vs テイカー** の経済設計

**欠点**:
- オンチェーンで order book を動かすには **高頻度 tx** が必要
- Ethereum では **ガス代で破綻** する
- Solana だから成立

### 3.3 ハイブリッド

**Raydium** は AMM + CLOB ハイブリッド:
- 自身のプールで価格決定
- **Serum (当時)、OpenBook** の order book にもルーティング
- 深みを両者から吸う

Phoenix、OpenBook は **純粋 CLOB**。機関向け。

### 3.4 Ethereum で CLOB が育たなかった理由

- **ガス代**: 注文を置く・キャンセルするたびに tx（$5-20）
- **遅延**: 12 秒ブロックでは HFT 不可
- **MEV**: sandwich attack で先回りされる

**dYdX v3** は CLOB を目指したが、後に **Cosmos SDK のアプリチェーン** に移行。

Ethereum L2 では GMX、Vertex などが試みているが、**Solana CLOB の勢いには及ばない**。

### 3.5 どちらが優れる？

**用途次第**:
- 長期 LP 提供、散歩的スワップ → **AMM (Uniswap)**
- HFT、プロフェッショナル取引、tight spread → **CLOB (Phoenix, OpenBook)**
- 平均的ユーザー → **Jupiter aggregator が両方使う**

## 4. Uniswap vs Raydium vs Phoenix

### 4.1 Uniswap (Ethereum)

- **TVL**: $5B+ (L1 v3)、L2 合わせて $8B+
- **Daily Volume**: $1-2B
- **Fee tier**: 0.05%, 0.3%, 1%
- **特徴**: 
  - v3 concentrated liquidity
  - v4 で hooks (カスタマイズ)
  - UNI トークン (ガバナンス)
- **運営**: Uniswap Foundation + Labs

### 4.2 Raydium (Solana)

- **TVL**: $1-2B
- **Daily Volume**: $1-3B (時期により Uniswap を超える)
- **Fee tier**: 0.25% (主流)、CLMM は可変
- **特徴**:
  - **CLMM** (Concentrated Liquidity Market Maker、Uniswap v3 相当)
  - **CPMM** (Constant Product、v2 相当)
  - **Standard AMM** (OpenBook order book 連携)
  - RAY トークン
- **運営**: Raydium DAO

### 4.3 Phoenix (Solana)

- **TVL**: 比較的小（CLOB は LP が違う意味）
- **特徴**:
  - **純粋 on-chain CLOB** (Serum の思想継承)
  - **tight spreads**
  - 機関・botter 向け
  - **エクストラ設計**: rent-exempt、無 crank（Serum の弱点を解決）
- **運営**: Ellipsis Labs

### 4.4 並べて見ると

| 項目 | Uniswap | Raydium | Phoenix |
|---|---|---|---|
| 方式 | 純 AMM | AMM + CLOB ハイブリッド | 純 CLOB |
| LP モデル | Concentrated | AMM + Order book | Maker |
| 1 swap コスト | $5-30 (L1) / $0.05-0.5 (L2) | $0.001-0.01 | $0.001-0.01 |
| 運営形態 | DAO | DAO | 会社 |
| 主要通貨 | ETH, USDC, USDT | SOL, USDC, USDT, meme | SOL, USDC |

### 4.5 Jupiter によるアグリゲーション

Solana では **Jupiter** が Raydium、Phoenix、Orca、Meteora、OpenBook、Lifinity などを横断:

- ユーザーは 1 画面
- 内部で best route 探索
- **Fee は各 DEX に**

Jupiter 自体は **1 swap で複数 DEX を経由** する機能（split route）も。

Ethereum にも 1inch、CowSwap、OpenOcean などがあるが、Jupiter ほどの支配力はない。

## 5. TVL と市場シェア

### 5.1 2022-2026 の推移

```
2022 Q1: Ethereum $80B+ / Solana $10B+ (DeFi TVL)
2022 Q4: 大暴落 (LUNA, FTX) → 両方大幅減
2023: Ethereum 横ばい / Solana 壊滅的（FTX 関連）
2024: Solana 復活 → $8B 超
2024 Q4: Solana の DEX volume が Ethereum を上回る月
2025-2026: Solana TVL $10-15B で安定、DEX volume で互角
```

### 5.2 DEX Volume トップチェーン

2025 年月次 DEX volume (DeFiLlama 風):
- **Solana**: $200-400B/month
- **Ethereum**: $150-300B/month
- **BSC**: $80-150B/month
- **Arbitrum**: $50-100B/month

**Solana が DEX volume で Ethereum を超えた** のは 2024 年の大きなニュース。

### 5.3 TVL (預かり資産)

2026 年時点:
- **Ethereum L1**: $50-80B
- **Ethereum L2s**: $30-50B
- **Solana**: $10-15B
- **BSC**: $5-8B
- **Avalanche**: $2-3B

**Ethereum が依然として預かり資産で圧倒的**。Solana は **volume 型、Ethereum は TVL 型** の傾向。

### 5.4 ユーザーの行動の違い

**Ethereum ユーザー**:
- 大きな資産を長期 hold
- DeFi で利回り狙い
- NFT コレクター

**Solana ユーザー**:
- 頻繁に取引
- ミームコイン・トレード
- NFT mint が手軽

## 6. MEV

### 6.1 Ethereum の MEV

**MEV (Maximal Extractable Value)**:
- validator (block proposer) が tx の順序を操作して得られる価値
- **front-running、sandwich attack、アービトラージ**

Ethereum の MEV:
- 2020 年から顕在化
- **Flashbots** が MEV-auction を導入
- **PBS (Proposer-Builder Separation)** で proposer と builder を分離
- MEV-Boost を使う validator が大半
- **検閲懸念** (OFAC 準拠ブロック)

年間 MEV 規模: **$1B 以上**

### 6.2 Solana の MEV

**Solana の MEV**:
- リーダーが明示的 → 予測可能
- **Jito** が Solana の MEV インフラ
- Jito 経由で bundle auction（Flashbots の Solana 版）
- **Priority fee** で順序が決まる

年間 MEV 規模: 成長中、$500M 以上

### 6.3 検閲耐性

**Ethereum**:
- MEV-Boost の relayer が OFAC 制裁対象 tx を除外していた時期あり
- → **検閲耐性が弱まった** と批判
- 現在は mev-boost-relay の多様化で改善中

**Solana**:
- Jito が rely している centralized infra あり
- 一時的に検閲の可能性

どちらも完璧ではない。

### 6.4 ユーザー体験への影響

**Ethereum**:
- Uniswap で large swap → **sandwich で数 bp 取られる**
- Cowswap などが対策済み

**Solana**:
- 通常スワップでも MEV bot が先回り
- Jupiter が **私的 tx 送信** で対策
- **Landed tx の 30-50% が MEV bot** の時期も

## 7. ステーキングと利回り

### 7.1 ステーキング利回り

**Ethereum**:
- **Liquid Staking (Lido 等)**: 3-4% APR
- **Restaking (EigenLayer)**: +1-3% 追加
- **Vanilla ステーキング**: 3-4%
- Total: 3-7%

**Solana**:
- **Vanilla ステーキング**: 6-8% APR
- **Liquid Staking (Jito 等)**: 7-9%
- MEV込で 7-10%

**Solana の方が利回りが高い** (インフレが高いため)。

### 7.2 Liquid Staking 市場

**Ethereum**:
- **Lido**: $30B+ TVL (ETH の 30%)
- **Rocket Pool**: $3B
- **EtherFi**: $8B (restaking LST)
- **独占懸念**: Lido が 30% を占めることへの議論

**Solana**:
- **Jito**: $5B+ TVL (SOL の 40%)
- **Marinade**: $2B
- **BlazeStake**: $1B
- LST 市場が急成長

### 7.3 DeIn への関係

DeIn は **Aave でステーキング報酬** を運用。これは L1 Ethereum 上で、**Lido 等の LST** を使う文化が成熟しているから。

Solana で同様の設計をするなら:
- **Jito で SOL → JitoSOL**
- **Kamino や MarginFi で貸出**

可能だが、エコシステムの成熟度が違う。

## 8. Bridge と相互運用性

### 8.1 主要 Bridge

**ETH ↔ Solana**:
- **Wormhole**: 最大手、Pyth ネットワークを運営
- **deBridge**: 高速、intent-based
- **LayerZero**: 2026-Solana 対応強化
- **Allbridge**: マイナー
- **Circle CCTP**: USDC のネイティブブリッジ

### 8.2 Bridge ハックの歴史

**Wormhole 2022-02**: $320M ハック（過去最大級）
**Ronin 2022-03**: $625M
**Nomad 2022-08**: $190M

Bridge は **DeFi 史上最も危険なコンポーネント**。

### 8.3 Native USDC の重要性

Circle の **Cross-Chain Transfer Protocol (CCTP)**:
- USDC を **burn + mint** で移動
- bridge contract に預けない → ハック耐性 ◎
- Ethereum、Solana、Arbitrum、Base、Optimism などに対応

**DeIn のような実用プロジェクトでは CCTP 経由の USDC 資金移動** が現実的。

### 8.4 クロスチェーン DeFi の将来

- **Intent-based** (CowSwap, UniswapX, Across): ユーザーは結果だけ指定
- **Chain abstraction**: ユーザーはチェーンを意識しない
- **Messaging (LayerZero, Wormhole Queries, Chainlink CCIP)**: データの相互利用

将来は **「どのチェーンを使うか」をユーザーが意識しない** 世界へ。

## 9. Q&A

### Q1: Solana の DeFi は安全？

**Ethereum ほど成熟していない**。ハック事例も多い (Mango Markets 2022 $100M、Saber など)。**Kamino、Drift、Jupiter、Raydium は比較的監査済み**。

### Q2: Ethereum L2 vs Solana、どちらで DeFi する？

- **資金が大きい、長期 hold**: Ethereum L1 or Arbitrum
- **頻繁なトレード、コスト重視**: **Solana**
- **新しいプロトコル試す**: Solana の方が多い

### Q3: MEV 対策されている DEX は？

**Ethereum**: CowSwap, UniswapX (intent-based で MEV protection)
**Solana**: Jupiter (private tx), Phoenix (order book で sandwich 不能)

### Q4: 日本円や JPYC はどちらにある？

- **JPYC (Ethereum)**: あり、Polygon にもあり
- **Solana**: Circle の **PYUSD** が主流、JPYC はまだ

### Q5: DeIn の Aave 運用を Solana でやるには？

**Kamino** または **MarginFi** が Aave 相当。しかし:
- **Aave V3 のセキュリティ実績**
- **JLP (Jupiter Perps LP)** で SOL → USDC の利回り
- 選択肢は限定的

Part 8 で詳述。

### Q6: pump.fun のような現象は Ethereum で起きる？

**L1 Ethereum では不可能**（gas 高すぎ）。**Base チェーン** で類似の現象 (friend.tech、ミームコイン) あるが、Solana ほど爆発していない。

## 10. まとめ

### 本記事で学んだこと

- **Ethereum DeFi**: **AMM 文化** (Uniswap)、レゴブロック、$100B+ TVL
- **Solana DeFi**: **AMM + CLOB ハイブリッド** (Raydium, Phoenix)、**Jupiter 集約**、$10-15B TVL
- **ガス代の違い**が AMM vs CLOB の選択を決めた
- **DEX volume** は 2024 以降 **Solana が Ethereum を抜く月** が出現
- **ステーキング利回り**: Ethereum 3-4%、Solana 6-8%
- **Bridge**: Wormhole、CCTP が主要。ハック事例多数

### 次の記事（Part 7）へ

次回は **L2・スケーリング戦略**。Ethereum の **Rollup-Centric Roadmap**、EIP-4844 以降の L2、Solana の **Firedancer・SVM L2 (Eclipse)** を比較。両チェーンが「**いかにスケールするか**」の戦略を対比する。

### 3行サマリ

- **Ethereum DeFi**: AMM (Uniswap) 中心、**$100B+ TVL**
- **Solana DeFi**: **AMM + CLOB ハイブリッド**、Jupiter aggregator が要、**DEX volume で Ethereum を超える月** も
- ガス代の違いが **AMM vs CLOB の生態系** を決めた

---

## 参考文献

- DeFiLlama. *Protocol Rankings and TVL.* [https://defillama.com/](https://defillama.com/)
- Uniswap Docs. [https://docs.uniswap.org/](https://docs.uniswap.org/)
- Raydium Docs. [https://docs.raydium.io/](https://docs.raydium.io/)
- Phoenix Docs. [https://ellipsislabs.xyz/phoenix](https://ellipsislabs.xyz/phoenix)
- Jupiter Aggregator. [https://jup.ag/](https://jup.ag/)
- Flashbots. *MEV-Boost.* [https://flashbots.net/](https://flashbots.net/)
- Jito. *MEV on Solana.* [https://www.jito.network/](https://www.jito.network/)
- 21Shares. *How Raydium and Jupiter Power Solana DeFi.* 2025.
