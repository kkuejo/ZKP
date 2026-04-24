---
title: "[Ethereum vs Solana 徹底比較 4/8] EVMとSealevel — スマートコントラクトの別の考え方"
emoji: "🧩"
type: "tech"
topics:
  - ethereum
  - solana
  - evm
  - sealevel
  - anchor
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事は両チェーンの **スマートコントラクト実行モデル** を深掘りする。Ethereum の **EVM** は「**コード + ストレージ一体型**」。コントラクトが作られると、その内部に mapping や variable が含まれる。一方 Solana の **SVM + Sealevel** は「**Program (コード) と Account (データ) が完全分離**」。Program はステートレス、データはすべて Account 側にある。この違いが **デプロイ・アップグレード・アップグレード可能性・開発者体験** すべてを変える。本記事では **(1) EVM の実行モデル**、**(2) Solidity の書き方**、**(3) SVM と Sealevel**、**(4) Program vs Account 分離**、**(5) Rust / Anchor での書き方**、**(6) 典型的アプリを両方で実装比較**、**(7) 開発者体験の差** を扱う。

## 0. 本記事の位置づけ

Part 3 で「Ethereum は直列、Solana は並列」の基礎を見た。本記事では **スマートコントラクト** の内部に踏み込む。

*Mastering Ethereum* で Solidity を学んだ読者にとって、Solana の「**プログラムに state が無い**」という考え方は最初衝撃的。しかしこれが並列性・Upgrade 性・セキュリティ特性のすべてを決める。

構成:

- **第1章**: EVM 実行モデル
- **第2章**: Solidity のアカウント・ストレージ構造
- **第3章**: SVM と Sealevel
- **第4章**: Solana の Program vs Account 分離
- **第5章**: Rust + Anchor
- **第6章**: 同じ DApp の両方実装
- **第7章**: 開発者体験の比較
- **第8章**: Q&A とまとめ

## 1. EVM の実行モデル

### 1.1 EVM の構造

**EVM** (Ethereum Virtual Machine):
- **スタックベース** VM（256 bit ワード）
- **永続ストレージ**: 各コントラクトに **mapping (256bit key → 256bit value)** が付随
- **メモリ**: 実行中の一時領域（byte addressable）
- **命令セット**: PUSH, POP, ADD, SSTORE, SLOAD, CALL, CREATE...

### 1.2 アカウントとコントラクト

Ethereum の **2 種類のアカウント**:

| 種類 | 持つもの |
|---|---|
| **EOA** (Externally Owned Account) | 残高のみ、秘密鍵で制御 |
| **Contract Account** | 残高 + **code** + **storage** |

コントラクトは **code と storage が分離不可**。アドレスが同じで内容が紐づく。

### 1.3 コントラクトのデプロイ

```solidity
// デプロイ
contract MyContract {
    uint256 public value;
    mapping(address => uint256) public balances;
    
    function setValue(uint256 _v) public {
        value = _v;
    }
}
```

- **デプロイ時**: bytecode と初期 storage がアドレスに固定される
- **アップグレード**: デフォルトでは **不可能**（Immutable）
- **Proxy パターン** で後付け可能（OpenZeppelin）

### 1.4 実行モデル

ユーザが `setValue(42)` を呼ぶと:
1. EVM がコントラクトの bytecode を読む
2. 引数をスタックに積む
3. 命令を順次実行
4. SSTORE 命令でストレージに書き込み
5. gas 消費、ブロックに tx が取り込まれる

**重要**: storage 書き込みは **最も gas コストが高い** (20,000 gas/key 初回)。

### 1.5 EVM の強みと弱み

**強み**:
- **シンプルな思考モデル**（「1 つのクラスのオブジェクト」的）
- **豊富なツール** (Hardhat、Foundry、OpenZeppelin)
- **15 年間のセキュリティ知見**

**弱み**:
- **並列化不可**
- **ストレージ肥大化**
- **アップグレード複雑** (Proxy の罠)

## 2. Solidity のアカウント・ストレージ構造

### 2.1 典型的な ERC-20

```solidity
contract MyToken {
    string public name = "MyToken";
    string public symbol = "MTK";
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    function transfer(address to, uint256 amount) public returns (bool) {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}
```

**構造**:
- **コントラクト = 1 つのアドレス**
- `balanceOf` mapping は **そのコントラクトの storage slot 1** に住む
- **全ユーザーの残高が同じ storage ツリー** に入る

### 2.2 Storage Slot

`balanceOf[address]` の storage slot は:

```
slot = keccak256(abi.encode(address, 1))
```

（1 は変数宣言順）

すべてのユーザー残高は **コントラクトの MPT** に詰まっている。結果として:
- **tx が同じコントラクトを触ると必ず競合**
- Uniswap の 1 プールに 1000 人が swap → **全部直列**

### 2.3 DeIn の例

DeIn の `IssuerManager.sol`:

```solidity
contract IssuerManager {
    mapping(uint256 => CertificateData) public certificates;
    mapping(uint256 => uint8) public certificatePrefecture;
    // ...
}
```

すべての NFT 契約データが **IssuerManager コントラクトの storage** に集まる。並列化は不可。

## 3. SVM と Sealevel

### 3.1 Solana VM (SVM)

**SVM** の基本:
- **sBPF** (Solana Berkeley Packet Filter) bytecode
- **レジスタベース** VM（EVM はスタックベース）
- **命令が EVM より高レベル** (Rust/C で書ける)
- **並列実行可能**

### 3.2 BPF の由来

Linux kernel で使われる **BPF** が元。ネットワークパケットフィルタリング用の軽量 VM。

Solana はこれを改造した **sBPF** を採用:
- Signed / unsigned 64bit 整数
- RAM access が制限されている（安全性）
- sBPF コンパイラが Rust / C コードから生成

### 3.3 実行ユニット: Compute Unit

EVM の「gas」に相当。
- 1 tx の compute unit 上限: **1.4M CU**
- 1 block の compute unit 上限: **48M CU**
- CU が不足すると tx 失敗

### 3.4 Sealevel との関係

- **SVM**: 単一 tx の実行 VM
- **Sealevel**: **複数 tx の並列スケジューリング**

Sealevel は SVM を多数並列で動かす司令塔。

## 4. Solana の Program vs Account 分離

### 4.1 Program = ステートレス

Solana の **Program**:
- **コード本体のみ** (BPF bytecode)
- **state を持たない**
- 「純粋関数」として入力アカウントを受け取って書き換える

### 4.2 Account = ストレージ

データはすべて **Account** 側に:
- User ごとの残高 → **個別の Account**
- NFT の所有権 → **個別の Account**
- DEX のプール状態 → **個別の Account**

Program は「**どの Account を触るか**」指示されて実行。

### 4.3 典型的なデータフロー

Solana 版 ERC-20 (SPL Token):

```
Token Program (コード、全 SPL トークン共通):
    transfer(src_account, dst_account, amount):
        src_account.balance -= amount
        dst_account.balance += amount

User A の USDC Account: { balance: 100, owner: TokenProgram, ... }
User B の USDC Account: { balance: 50, owner: TokenProgram, ... }

Transaction:
    program: Token Program
    accounts: [User A の USDC, User B の USDC]
    instruction: transfer(10)
```

実行すると:
- Program は自身の bytecode を実行
- 渡された accounts の data を書き換え
- Token Program 自体は **不変** (state が無いから)

### 4.4 Account Owner の意味

各 Account には **`owner` フィールド** = 「どの Program が書き換え権を持つか」

- **System Program が owner**: 普通の SOL account
- **Token Program が owner**: SPL トークン account
- **あなたの DApp が owner**: DApp 専用 data account

**owner でないプログラムは書き換え不可**。これがセキュリティの基盤。

### 4.5 PDA (Program Derived Address)

Program が管理する account をプログラム的に生成:

```rust
let (pda, bump) = Pubkey::find_program_address(
    &[b"user_vault", user_pubkey.as_ref()],
    &program_id,
);
```

**PDA = ユーザー秘密鍵ではなく Program が署名権を持つ account**。ユーザーごとに deterministic に生成できる。

### 4.6 Upgrade 性

Program は **アップグレード可能** (BPF Upgradeable Loader):
- コードを差し替え可能（権限者が）
- 既存の Account data はそのまま
- **Ethereum の Proxy よりシンプル**

ただし、コード変更で既存の data 構造が合わなくなるリスクは開発者の責任。

## 5. Rust + Anchor

### 5.1 Solana 開発の現実

純粋な Rust で Solana program を書くと:
- Account パース、serialize/deserialize 全部自前
- エラーが起きやすい
- コード量が多い

そこで **Anchor** フレームワークを使う。

### 5.2 Anchor の例

```rust
use anchor_lang::prelude::*;

declare_id!("FvF...");

#[program]
pub mod my_token {
    use super::*;
    
    pub fn initialize(ctx: Context<Initialize>, amount: u64) -> Result<()> {
        let account = &mut ctx.accounts.user_account;
        account.balance = amount;
        Ok(())
    }
    
    pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {
        let from = &mut ctx.accounts.from;
        let to = &mut ctx.accounts.to;
        
        require!(from.balance >= amount, CustomError::InsufficientBalance);
        from.balance -= amount;
        to.balance += amount;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub user_account: Account<'info, UserAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Transfer<'info> {
    #[account(mut)]
    pub from: Account<'info, UserAccount>,
    #[account(mut)]
    pub to: Account<'info, UserAccount>,
    pub authority: Signer<'info>,
}

#[account]
pub struct UserAccount {
    pub balance: u64,
}
```

### 5.3 Anchor が解決すること

- **アカウント検証**: `#[derive(Accounts)]` で安全にパース
- **Serialize/Deserialize**: `#[account]` で自動
- **IDL (Interface Description Language)**: クライアントコードを自動生成
- **テストフレームワーク**: Rust + TypeScript

### 5.4 Rust vs Solidity の学習曲線

- **Solidity**: 数週間で基礎、数ヶ月で実戦
- **Rust + Anchor**: 数ヶ月で基礎、半年〜1年で実戦

**理由**:
- Rust の所有権・ライフタイム
- Solana のアカウントモデルの特殊さ
- PDA、CPI (Cross-Program Invocation) などの概念

*Mastering Ethereum* を読んだ読者が Solana を学ぶと「**また初学者に戻った**」感覚になる。

## 6. 同じ DApp の両方実装

### 6.1 シンプルなカウンターDApp

**Ethereum (Solidity)**:

```solidity
contract Counter {
    uint256 public count;
    
    function increment() public {
        count += 1;
    }
}
```

**Solana (Rust + Anchor)**:

```rust
#[program]
pub mod counter {
    use super::*;
    
    pub fn init(ctx: Context<Init>) -> Result<()> {
        ctx.accounts.counter.count = 0;
        Ok(())
    }
    
    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        ctx.accounts.counter.count += 1;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Init<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
}

#[account]
pub struct Counter {
    pub count: u64,
}
```

### 6.2 観察ポイント

**Ethereum**:
- 1 つの contract に `count` が内包される
- `new Counter()` で deploy、以後アドレス指定
- すべてのユーザーが同じカウンターを触る

**Solana**:
- `Counter` struct が **Account に格納される**
- ユーザーごと/用途ごとに別の Counter Account を作れる
- **並列**に動く (別の Counter なら干渉しない)

### 6.3 複数ユーザーのカウンター

Ethereum で「ユーザーごとのカウンター」を作るには:

```solidity
mapping(address => uint256) public counters;
// counters[msg.sender]++;
```

Solana で同じことは:

```rust
// 各ユーザー用の PDA を作る
let (pda, _) = Pubkey::find_program_address(&[b"counter", user.key().as_ref()], &program_id);
// ctx.accounts.counter がその PDA
```

**Solana は PDA で自然に分離され、並列で動く**。

### 6.4 本格的な DApp

Uniswap V2 相当を両方で書いてみると:
- **Solidity**: 200-300 行
- **Anchor Rust**: 500-800 行（account 定義、CPI 含む）

Solana の方がコード量が多いが、**並列性・アップグレード性・セキュリティ** が構造的に保証される。

## 7. 開発者体験の比較

### 7.1 ツール

| カテゴリ | Ethereum | Solana |
|---|---|---|
| フレームワーク | Hardhat, Foundry | Anchor |
| テスト | Mocha, Foundry | Mocha (+ Anchor CLI) |
| デプロイ | hardhat deploy, forge create | anchor deploy |
| クライアント | ethers.js, viem, wagmi | @solana/web3.js, Anchor Client |
| ウォレット統合 | MetaMask, WalletConnect | Phantom, Solana Wallet Adapter |
| Explorer | Etherscan | Solscan, Solana Explorer |

### 7.2 学習リソース

**Ethereum**:
- *Mastering Ethereum*
- Solidity Docs、CryptoZombies
- Ethernaut、Damn Vulnerable DeFi (security)
- 無限の YouTube / Blog

**Solana**:
- *Solana Cookbook*
- Anchor Book
- Buildspace, Solana Bootcamp
- 相対的に少ないが急速に拡充中

### 7.3 デバッグ

**Ethereum**:
- **Foundry の trace** が非常に強力
- **Tenderly** で本番の tx を simulation & debug

**Solana**:
- **solana logs** でプログラムログ
- **Anchor CLI** の simulate
- **Explorer での tx inspection**
- デバッグは依然として Ethereum より困難

### 7.4 セキュリティ

**Ethereum**:
- Re-entrancy、integer overflow、front-running の知見が蓄積
- **OpenZeppelin** のテンプレートで大半をカバー
- 監査会社が多数 (Consensys Diligence、Trail of Bits、OpenZeppelin Audit)

**Solana**:
- **account confusion attack**、**signer check 漏れ**、**PDA hijack** など独自の脆弱性
- **Anchor** が多くの典型的ミスを防ぐ
- 監査会社も増加中 (OtterSec、Zellic、Neodyme)

### 7.5 個人的な所感

「**どちらで開発すべきか**」は:

- **既存エコシステム活用** → Ethereum/EVM 互換
- **UX 重視** → Solana
- **低コスト大量 tx** → Solana
- **長期インフラ** → Ethereum
- **学習段階** → Ethereum から入り、Solana を後で学ぶ

## 8. Q&A

### Q1: Solana で ERC-20 相当を書くには？

**SPL Token** を使う。自前実装不要、Token Program を呼ぶだけ。EVM の ERC-20 を毎回自作する文化とは違う。

### Q2: Program が state を持たないの、不便では？

**慣れると自然**。むしろ「**すべての state が Account として可視化**」される分、設計が明示的になる。難しいのは最初の発想転換。

### Q3: Anchor が標準なら他のフレームワークは？

- **Pinocchio**: 軽量・低レベル
- **Steel**: シンプル志向
- **Native Rust**: フル制御

Anchor が 80% のシェア。

### Q4: Cross-Program Invocation (CPI) って？

Ethereum の `.call()` に相当。**Program から別の Program を呼ぶ**。**PDA で署名** することで、Program が自律的に他 Program を呼べる。DeFi の積み重ね (composability) の核。

### Q5: Solidity を書ける人が Solana を始めるには？

1. **Rust の基礎** (The Rust Book の 1-6 章)
2. **Solana Cookbook** で基本概念
3. **Anchor Book** でフレームワーク
4. **Mini-program** を書く（Counter、Escrow、Token）

3-6 ヶ月で一通り書けるようになる。

### Q6: アップグレード可能な Program は安全？

**慎重な設計が必要**。権限者が change できるということは、**権限者を乗っ取れば全資金を持ち去れる**。Multisig で制御するのが定石。

## 9. まとめ

### 本記事で学んだこと

- **Ethereum EVM**: スタックベース、**コード + ストレージ一体型**、直列実行
- **Solana SVM + Sealevel**: レジスタベース、**Program (コード) と Account (データ) 完全分離**、並列実行
- **事前アカウント宣言** が Solana の並列実行を支える
- **Solidity** はオブジェクト指向的、**Rust + Anchor** は関数型・データ駆動的
- **Anchor** が Solana 開発の標準フレームワーク
- **学習曲線**: Solidity < Rust + Anchor。しかし一度マスターすれば両方で強力

### 次の記事（Part 5）へ

次回は **ネットワーク障害耐性とダウンタイム**。Solana はなぜ過去に止まったか、Ethereum のクライアント多様性戦略、そして Firedancer の意義。**「分散性のコスト」** を実例で見る。

### 3行サマリ

- **Ethereum**: **コード + ストレージ一体**、直列実行、Solidity で書く
- **Solana**: **Program (ステートレス) + Account (データ) 分離**、並列実行、Rust/Anchor で書く
- **学習曲線は Solana が急**、しかし並列性・低ガスで **新時代の UX** を実現

---

## 参考文献

- Gavin Wood. *Ethereum: A Secure Decentralised Generalised Transaction Ledger.* Yellow Paper, 2014.
- Solana Docs. *Sealevel: Parallel Processing Thousands of Smart Contracts.*
- Anchor Book. [https://book.anchor-lang.com/](https://book.anchor-lang.com/)
- Solana Cookbook. [https://solanacookbook.com/](https://solanacookbook.com/)
- Paul Berg. *Foundry Book.* [https://book.getfoundry.sh/](https://book.getfoundry.sh/)
- OpenZeppelin. *Contracts Wizard.* [https://docs.openzeppelin.com/contracts/](https://docs.openzeppelin.com/contracts/)
- Solana Docs. *EVM to SVM: Accounts.* [https://solana.com/developers/evm-to-svm/accounts](https://solana.com/developers/evm-to-svm/accounts)
