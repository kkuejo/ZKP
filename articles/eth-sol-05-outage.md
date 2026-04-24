---
title: "[Ethereum vs Solana 徹底比較 5/8] 障害耐性 — Solanaはなぜ止まるのか、Ethereumはなぜ止まらないのか"
emoji: "🚨"
type: "tech"
topics:
  - ethereum
  - solana
  - outage
  - firedancer
  - client
published: false
---

**日付**: 2026年4月24日
**学習内容**: 「**Solana は落ちる**」という言葉は、2022-2023 年の実情を反映した批判だった。Ethereum は 2015 年の起動以来、**メインネットが完全停止した歴史的事故が基本ゼロ**。一方 Solana は過去 **8 回程度** の完全ダウンタイム。この差は、両チェーンの **設計思想と実装戦略** の違いから必然的に生じる。本記事では **(1) Solana の主要ダウン事例**、**(2) 構造的な原因**、**(3) Ethereum のクライアント多様性戦略**、**(4) 2024 年 2 月の最後の完全停止と以降の安定性**、**(5) Firedancer の意義**、**(6) バリデータ構成・ハードウェア要件の差**、**(7) 金融インフラとしての信頼性** を扱う。

## 0. 本記事の位置づけ

Part 1-4 で「**Solana は性能優先**、**Ethereum は分散優先**」と見てきた。この「**性能のコスト**」は何で払われるのか？

答え: **ダウンタイム**、**中央集権化リスク**、**金融インフラとしての不安定さ**。

逆に、**Ethereum が遅いおかげで、止まらない**。この冷徹な関係を本記事で見る。

構成:

- **第1章**: Solana のダウンタイム年表
- **第2章**: 障害の構造的原因
- **第3章**: Ethereum のクライアント多様性
- **第4章**: Ethereum のダウン事例
- **第5章**: 2024 年以降の Solana 安定性
- **第6章**: Firedancer の意義
- **第7章**: バリデータ構成
- **第8章**: Q&A とまとめ

## 1. Solana のダウンタイム年表

### 1.1 主要な完全停止事故

| 日付 | 長さ | 原因 |
|---|---|---|
| 2021-09-14 | **17 時間** | Raydium IDO で tx スパム、メモリ不足 |
| 2022-01-21 | 28 時間（部分的） | 高負荷でフォーク頻発 |
| 2022-04-30 | 7 時間 | NFT mint ボットで 120 万 tx/s、vote 不達 |
| 2022-05-01 | 継続 | 同上の継続 |
| 2022-06-01 | 5 時間 | durable nonce tx のバグ |
| 2022-09-30 | 7 時間 | Misconfigured validator |
| 2023-02-25 | 19 時間 | テスト機能の本番流出 |
| **2024-02-06** | **5 時間** | **JIT cache 無限ループ** |
| **2024-02-06 以降** | **なし** | 安定期に突入 |

### 1.2 Ethereum との比較

Ethereum は **2015 年のローンチ以降、L1 が完全停止した事例が基本ゼロ**:
- 2020 年に **Infura がダウン** (Geth のバグ) で一部 dApp 停止、しかしチェーン自体は動いていた
- 2021 年に **Geth consensus bug** で short reorg、しかし停止ではない

**Ethereum は「止まらない」ことを最優先** に設計されている。

### 1.3 数値化すると

2021-2023 の累計ダウンタイム:
- Solana: **約 4 日**（99% uptime 目標に対し 98.7%）
- Ethereum: **ほぼ 0 分**（99.99% 超）

金融インフラとしては **桁違い** の信頼性差。

## 2. 障害の構造的原因

### 2.1 共通の構造的要因

Solana のダウンの大部分は以下のいずれか:

1. **スパム tx によるリソース枯渇** (メモリ、CPU)
2. **実装のバグ** (複雑性の高さゆえ)
3. **Vote tx が届かない** (ネットワーク飽和)
4. **リーダーの選出失敗** (Tower BFT の脆さ)

### 2.2 根本原因: 性能最適化の代償

Solana は **最大性能を引き出すため**:
- **メモプールなし** (Gulf Stream) → スパム耐性弱い
- **シングルスレッド PoH** → リーダーが重い
- **高 TPS** → バグで大量 tx がチェーンに拡散
- **クライアント 1 種 (2024 まで)** → 全ノードが同じバグで落ちる

これらは **速度を得るため** の設計判断。片方が崩れるとチェーンが止まる。

### 2.3 「復旧がコミュニティ調整で行われる」問題

Solana の復旧は:
1. バリデータのチャットで議論
2. Snapshot 時刻を決める
3. 80% 以上のバリデータが同じ snapshot から **再起動**
4. コンセンサス再構築

これは **人的調整** を要する。Ethereum のような「止まらないので復旧不要」とは別世界。

### 2.4 Ethereum の安定性の源泉

**Ethereum の安定性** は:
- **クライアント多様性**: バグがあっても全ノードが同時に落ちない
- **バリデータ数**: 100 万超、地理的分散
- **保守的な設計**: 性能より安定を選ぶ

逆に言えば、**遅いからこそ止まらない**。

## 3. Ethereum のクライアント多様性

### 3.1 実行クライアント (Execution Layer)

EVM 実行を担うソフト。**複数独立実装**:

| クライアント | 言語 | 開発者 | シェア |
|---|---|---|---|
| **Geth** | Go | Ethereum Foundation | 約 54% |
| **Nethermind** | C# | Nethermind | 約 21% |
| **Besu** | Java | ConsenSys | 約 13% |
| **Erigon** | Go | Erigon Team | 約 8% |
| **Reth** | Rust | Paradigm | 約 4% (急拡大中) |

**同じ Ethereum 仕様を独立実装** → 1 つにバグがあっても他が動く。

### 3.2 コンセンサスクライアント (Consensus Layer)

PoS のコンセンサスを担うソフト。これも複数:

| クライアント | 言語 | シェア |
|---|---|---|
| **Prysm** | Go | 約 40% |
| **Lighthouse** | Rust | 約 30% |
| **Teku** | Java | 約 14% |
| **Nimbus** | Nim | 約 8% |
| **Lodestar** | TypeScript | 約 5% |
| **Grandine** | Rust | 約 3% |

### 3.3 なぜこれほど多くの実装？

**設計哲学**: 「**2/3 以上のバリデータが同じクライアントを使わない**」ことが目標。

理由:
- 1 クライアントに致命的バグ → 2/3 超なら **誤ってファイナライズ** する恐れ
- 複数実装があれば、1 つが落ちても 2/3 は動く

Vitalik 自身が「**クライアント多様性は非交渉的 (non-negotiable)**」と繰り返し発言。

### 3.4 ペナルティ

単一クライアント依存のバリデータは:
- **コミュニティから非推奨**
- バリデータモニタリングでシェア可視化 (clientdiversity.org)

### 3.5 実例: 2023 年 Nethermind バグ

2023 年に Nethermind のバグで同クライアントのバリデータが数時間オフライン。Geth、Besu、Erigon が動作していたので **Ethereum 自体は止まらず**。これが多様性の価値。

## 4. Ethereum のダウン事例（稀）

### 4.1 2020 年 11 月 Infura ダウン

**Infura** (Ethereum RPC プロバイダ) がダウン。
- チェーン自体は動いていた
- しかし **MetaMask・Coinbase・多くの dApp** が機能停止
- 原因: Geth クライアントのバグ
- **中央集権化されたインフラへの依存** を露呈

### 4.2 2021 年 Geth consensus bug

Geth の古いバージョンでコンセンサスバグ。
- **Chain split** が発生
- 数時間でアップグレード、両方のフォークが再統一
- **チェーンは止まらず**

### 4.3 2023 年 Teku bug

Teku のバグで同クライアントの validator が効率低下。
- ネットワーク全体への影響は軽微
- 他クライアントが健全だったので **no outage**

### 4.4 Shanghai / Cancun 等のアップグレード

すべてスムーズに進行。**ハードフォークで止まった歴史なし**（bitcoin と違って慎重）。

### 4.5 結論: 「**止まらないこと**」はデフォルト

Ethereum の 10 年近い運用で、**L1 完全停止は実質ゼロ**。これは単なる運ではなく、**設計と文化**の結果。

## 5. 2024 年以降の Solana 安定性

### 5.1 2024 年 2 月以降の安定期

2024-02-06 の JIT cache バグを最後に、**完全停止は発生していない** (本記事執筆時点 2026-04)。

= **2 年以上連続稼働**。過去最長。

### 5.2 安定化の要因

1. **Core engineering チームの拡大**: Anza (旧 Solana Labs) + Helius, Triton など
2. **Agave** (旧 Solana Labs クライアント) の改善
3. **Frankendancer** の部分導入（Firedancer の前段階）
4. **地震対策的な復旧ドリル** の習熟
5. **スパム対策**: Stake-weighted QoS, Priority fees

### 5.3 しかし完全ではない

"完全停止" はないが、**部分的な問題** は継続:

- **2024-04**: 数時間の高レイテンシ・低 TPS
- **2025-02**: ミームコイン FOMO で混雑、実質使えない状態
- 2024-10〜2025-02 の間に **9 件の非公式な障害** が外部モニタで検出

「止まらないが遅くなる」フェーズ。Ethereum の「**遅いが止まらない**」とは別種の不安定さ。

### 5.4 Firedancer への期待

Firedancer (後述) の完全導入で、残る構造的脆弱性が解消される見込み。

## 6. Firedancer の意義

### 6.1 Firedancer とは

**Jump Crypto** が開発する **独立実装の Solana クライアント** (C++)。

目的:
1. **性能向上** (目標 1M TPS)
2. **クライアント多様性** (第 2 の実装)

Jump Crypto は High-Frequency Trading 企業。**金融インフラ品質** の実装を目指す。

### 6.2 Frankendancer (部分実装)

2024-12 メインネット投入:
- Firedancer の **一部コンポーネント** (network stack、signature verify)
- 残りは Agave (既存 Solana Labs クライアント)
- **既に 20% の stake** で動作 (2025-10)

### 6.3 Full Firedancer

2025-2026 完成予定:
- **純粋 C++ 実装**、Agave から独立
- クライアント多様性を初めて達成
- スループット大幅向上

### 6.4 クライアント多様性への含意

Firedancer が広く採用されれば:
- Solana は **2 クライアント** 体制
- 単一バグで全停止するリスク激減
- **Ethereum 水準の耐障害性** に近づく

ただし Ethereum の **5+ クライアント** には及ばない。

### 6.5 批判

- **Firedancer も Solana 仕様のバグには対処できない**（仕様自体のバグなら両方落ちる）
- **2/3 超のバリデータが同じクライアントを使ってはいけない** ルールが Solana にはまだない
- **Jump Crypto が単一の開発元** という集中リスク

完全な多様性には道のり遠い。

## 7. バリデータ構成・ハードウェア要件

### 7.1 ハードウェア要件

**Ethereum (Consensus + Execution)**:
- CPU: 4 core
- RAM: 16 GB
- SSD: 2 TB
- Network: 25 Mbps 上下
- **家庭用 PC で十分**

**Solana (Validator)**:
- CPU: 16-24 core
- RAM: **128-256 GB**
- SSD: NVMe 2 TB × 2 (snapshot + ledger)
- Network: **10 Gbps**
- **データセンター級**

### 7.2 コスト

- **Ethereum 家庭ステーキング**: 電気代 + 初期 $1,500
- **Solana バリデータ**: 月 **$1,000-2,000** (vote tx 代 + インフラ)

### 7.3 参入可能性

- **Ethereum**: **誰でも可能** (32 ETH あれば)
- **Solana**: **機関・専業業者がほぼ独占**

### 7.4 バリデータ数

| チェーン | 総バリデータ | アクティブ |
|---|---|---|
| Ethereum | 約 **1,000,000** | 約 1,000,000 |
| Solana | 約 **1,400** | 約 1,400 |

700 倍以上の差。**分散性の指標として明白**。

### 7.5 地理的分散

Ethereum: 80+ 国
Solana: 約 40 国

どちらも主要国（US、ドイツ、フィンランド、シンガポール、日本）に集中。

## 8. Q&A

### Q1: Solana が止まって DeFi 資産が消える？

**消えない**。停止中は tx が通らないだけ。再開後は止まる前の状態から続行。しかし **清算ポジション** が動かせないのは問題（Ethereum で過去に MakerDAO で起きた類の話）。

### Q2: Ethereum は本当に止まらない？

**歴史的には止まっていない**。ただし理論的には、クライアント多様性が 2/3 を失うと停止する可能性あり。だからこそ diversity が絶対視される。

### Q3: Firedancer で Solana は「止まらない」になる？

**大きく改善する見込み**。ただし単一組織 (Jump Crypto) が作ったクライアントなので、Ethereum の 5+ 独立実装には及ばない。

### Q4: Web3 インフラとしての信頼性、どっちが上？

**インフラ品質としては現状 Ethereum が上**。金融インフラ、社会インフラには保守性が必要。Solana は「**高速だが稀に止まる**」ポジション（CDN や DNS に近い）。

### Q5: Solana を使うなら、障害対策は？

- **Treasury は止まっても大丈夫な設計**
- **オラクル取込は非同期 retry**
- **清算は止まらないチェーン（Ethereum 等）を併用**
- **ユーザーへの communicate** は別経路（Discord、Twitter）

### Q6: DeIn で障害耐性はどう考える？

DeIn は **地震発生後の自動支払い** が核。支払い遅延はシビア:
- **Ethereum**: 安定性 ◎、しかし gas 代高
- **Solana**: 低コスト ◎、しかし過去の障害歴
- **Ethereum L2**: バランス型

Part 8 で深掘り。

## 9. まとめ

### 本記事で学んだこと

- **Solana** は過去 **8 回程度の完全停止**、最後は 2024-02-06 の JIT cache バグ
- **Ethereum** は 2015 年ローンチ以降 **L1 完全停止ゼロ**
- 原因: Solana の性能最適化（メモプール無、シングルクライアント、高 TPS）が **障害に脆い**
- **Ethereum のクライアント多様性**: 5+ の独立実装、**2/3 超が同じクライアントを使わない** ことを非交渉的ルール
- **Firedancer** で Solana も多様性達成へ（ただし Ethereum 水準はまだ）
- **バリデータ参入**: Ethereum は家庭用 PC、Solana はデータセンター級

### 次の記事（Part 6）へ

次回は **DeFi エコシステム**。Uniswap vs Raydium、AMM vs CLOB、MEV 環境の違い、TVL の推移。両チェーンの **DeFi 文化** を比べる。

### 3行サマリ

- **Solana** は過去 8 回停止、**Ethereum** は基本ゼロ。**速度の代償として信頼性**
- **Ethereum のクライアント多様性 (5+)** が障害耐性の源泉、**Solana は 1-2**
- **Firedancer** で改善中だが、**金融インフラとしての信頼性は Ethereum が依然優位**

---

## 参考文献

- Helius. *Complete History of Solana Outages: Causes and Fixes.* [https://www.helius.dev/blog/solana-outages-complete-history](https://www.helius.dev/blog/solana-outages-complete-history)
- StatusGator. *Solana Outage History.* [https://statusgator.com/blog/solana-outage-history/](https://statusgator.com/blog/solana-outage-history/)
- clientdiversity.org. [https://clientdiversity.org/](https://clientdiversity.org/)
- Vitalik Buterin. *Client Diversity is Non-Negotiable.* 2022 blog posts.
- Firedancer GitHub. [https://github.com/firedancer-io/firedancer](https://github.com/firedancer-io/firedancer)
- Blockdaemon. *Firedancer Deep Dive.* 2024.
- CryptoSlate. *Firedancer Live, but Solana Still Violates Ethereum's Non-Negotiable Safety Rule.*
