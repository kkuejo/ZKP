# Aaveフォークテストのやり方と、2026年4月18日の$292M Kelp DAO事件

**日付**: 2026年4月20日
**学習内容**: DeInプロジェクトでは、ステーキングされたETHをAaveに預けて運用する設計を採用しており、その検証のために **Aave mainnetをAnvil上にフォーク** してテストを書いてきた。本日のエントリではその**Aaveフォークテストの一般的な手順**を整理する。そして2026年4月18日にKelp DAOで発生し、Aaveに $196M のbad debtを残した大規模exploit事件についてもまとめ、**DeInのステーキング設計を再考すべき論点**を抽出する。

## 0. 本日の位置づけ

DeInは「地震が起こらない期間のステーカーへの報酬原資」をAave V3 の ETH pool から得る設計になっている（ETH → Aave → aWETH → Treasury という資金フロー）。つまり**DeInは外部DeFiプロトコル（Aave）に資金を預けて運用するフロー**を持っており、Aaveの挙動を正確に再現できるテスト環境が必要になる。

そのために採用してきたのが **Mainnet Fork Testing**（mainnetの状態をローカルにコピーして実行する）である。[solidity-contracts/test/fork/FundManagement.fork.t.sol](solidity-contracts/test/fork/FundManagement.fork.t.sol) や [solidity-contracts/scripts/test-aave-fork.sh](solidity-contracts/scripts/test-aave-fork.sh) は、この方式でAave・Lido・Curveの本物のコントラクトに対して統合テストを回すためのもの。

一方で、2026年4月18日に **Kelp DAO の rsETH が $292M 盗まれ、そのうち $196M が Aave の bad debt として残る**という、2026年最大のDeFi事件が発生した。これはAaveそのもののコードが破られたわけではないが、**「Aaveに依存するプロトコル」全体に対する示唆が極めて大きい**。DeInはまさにその「Aaveに依存するプロトコル」なので、設計を見直すべきタイミングである。

本エントリは以下の2部構成:

- **前半（第1〜3章）**: Aaveフォークテストの一般的なやり方（DeInのコード実例を交えて）
- **後半（第4〜7章）**: 2026年4月18日の事件まとめと、DeInステーキング設計への示唆

## 1. フォークテストとは何か

**フォークテスト（Fork Testing）** とは、**既に本番チェーン（mainnet）上で稼働している状態を、ローカル環境にコピーして、そこに対して新規コントラクトを乗せてテストする**方法である。

### 1.1 なぜフォークテストが必要か

Aaveのような大規模DeFiプロトコルを呼び出すコントラクトをテストしたいとき、選択肢は3つある:

| 手法 | メリット | デメリット |
|---|---|---|
| **① モックで書く** | 速い・無料・再現性◎ | 本物と挙動がズレる危険（Aaveの内部数学は複雑） |
| **② testnet（Sepolia等）にデプロイして試す** | 本物に近い | Sepoliaに `aWETH` があっても流動性や金利カーブは本番と違う。セットアップ面倒 |
| **③ mainnetフォークテスト** | **本物のAave V3コントラクトを相手にテストできる**、履歴・流動性・金利もすべて本番 | RPCエンドポイントに依存（速度・料金） |

DeFi連携はモックだと**Aaveのインデックス計算・aTokenのリベース挙動・利息蓄積**などの微妙な振る舞いが再現できず、本番でいきなりバグる典型パターン。③が最も信頼性が高い。

### 1.2 仕組みのイメージ

```
[Ethereum Mainnet（本物）]
  ├─ Aave V3 Pool (0x8787...4E2)
  ├─ aWETH (0x4d5F...4E8)
  ├─ WETH (0xC02a...Cc2)
  └─ 何百万ものアカウント状態
         │
         │ ブロック番号Nの時点の状態を"スナップショット"
         │ フォーク元として参照（オンデマンドで読む）
         ▼
[ローカルAnvil]
  ├─ Aave V3 Pool ← 本物と同じアドレス・同じコード・同じstorage
  ├─ 自作Staking  ← ここに新しいコントラクトをデプロイ
  └─ 時間を進めたり、任意のアカウントでなりすましたりできる
```

キモは **「Aaveのコントラクトをローカルに再デプロイする必要がない」** こと。Anvilは `--fork-url` で指定されたmainnetから **必要になったストレージスロットだけ遅延読み込み（lazy load）** する。その上に自分で `new Staking(...)` すればmainnet上のAaveをそのまま呼べる。

## 2. Aaveフォークテストの一般的な手順

ここからは、DeInのコードを参照しながら一般的な手順を整理する。Foundryを使う前提。

### 2.1 Step 1: Aave V3のmainnetアドレスを押さえる

Aaveは各チェーン・各バージョンで固定アドレスを持っている。Ethereum Mainnet の Aave V3 の主要アドレスは:

| コントラクト | アドレス | 役割 |
|---|---|---|
| **Aave V3 Pool** | `0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2` | supply / withdraw / borrow / repay の中核 |
| **WETH Gateway** | `0xD322A49006FC828F9B5B37Ab215F99B4E5caB19C` | ETHのまま（ラップせずに）Aaveに預けるラッパー |
| **aWETH** | `0x4d5F47FA6A74757f35C14fD3a6Ef8E3C9BC514E8` | WETHを預けた証として受け取るrebasing token |
| **WETH** | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | 言わずもがな |

DeInでは [FundManagement.fork.t.sol の86〜93行目](solidity-contracts/test/fork/FundManagement.fork.t.sol#L86-L93) に `constant` で並べてある:

```solidity
address constant AAVE_POOL = 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2;
address constant WETH_GATEWAY = 0xD322A49006FC828F9B5B37Ab215F99B4E5caB19C;
address constant A_WETH = 0x4d5F47FA6A74757f35C14fD3a6Ef8E3C9BC514E8;
address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
```

このアドレスはAave公式の address-book から取る。**他のチェーン（Arbitrum、Optimism、Polygon等）でも Aave V3 はデプロイされているが、アドレスは違う** ので、フォーク先のチェーンに合わせて書き換える必要がある。

### 2.2 Step 2: RPCエンドポイントを用意する

フォーク元となる **mainnet へのRPCアクセス** が必要。選択肢:

| 選択肢 | 料金 | 注意点 |
|---|---|---|
| **Alchemy / Infura** の無料枠 | 無料（リクエスト制限あり） | フォークテストはRPCを大量に叩くので制限にひっかかりやすい |
| **Alchemy / Infura** 有料 | 月額 $49〜 | 開発中は有料枠推奨 |
| **Publicnode** 等の公開RPC | 無料 | レイテンシ・可用性は保証なし |
| **自前ノード** | 高コスト | ガチ勢向け |

DeInの [test-aave-fork.sh の8行目](solidity-contracts/scripts/test-aave-fork.sh#L8) は `https://ethereum-rpc.publicnode.com` を使っている:

```bash
anvil --fork-url "https://ethereum-rpc.publicnode.com" --chain-id 31337
```

Foundry側で `forge test --fork-url $ETH_RPC_URL` と渡す場合も同じ。環境変数 `ETH_RPC_URL` に入れておくのが慣例。

### 2.3 Step 3: 2通りのフォーク実行モード

フォークを使ったテストには**独立した2つの流派**がある。

#### モード A: Foundry内蔵フォーク（`forge test --fork-url`）

Foundryはネイティブに "テストランナー内でフォーク" できる:

```bash
forge test --match-contract FundManagementForkTest --fork-url $ETH_RPC_URL -vvv
```

この場合、各テスト関数の `setUp()` 時点で **毎回** mainnet状態を読み込み、`vm.deal` / `vm.warp` / `vm.startPrank` 等のチートコードを駆使してテストする。[FundManagement.fork.t.sol の setUp](solidity-contracts/test/fork/FundManagement.fork.t.sol#L124-L156) はまさにこれ。

**メリット**: テストごとに状態がリセットされる、Foundryチートコードがフル活用できる、CIに組み込みやすい

**デメリット**: RPCコールが多い（各テストでstorageを読む）、テスト時間がやや長い

#### モード B: Anvilを常駐させてスクリプトで叩く

`anvil --fork-url ...` をターミナルで起動しっぱなしにし、別ターミナルから `cast` や `forge script` で接続する:

```bash
# Terminal 1
anvil --fork-url "https://ethereum-rpc.publicnode.com" --chain-id 31337

# Terminal 2
forge script script/DeployStakingSystem.s.sol --rpc-url http://127.0.0.1:8545 --broadcast
./scripts/test-aave-fork.sh
```

[test-aave-fork.sh](solidity-contracts/scripts/test-aave-fork.sh) はまさにこれ。デプロイしたコントラクトアドレスをjsonから引っ張り出して、`cast send` / `cast call` でAave操作をトランザクションとして実行していく。

**メリット**: 実際のフロントエンド統合に近い形でE2Eが組める、状態を引き継いで連続操作ができる

**デメリット**: 状態をリセットしたければ再起動が必要、並列化しにくい

DeInは**両方用意**している。フォーク単体テスト（モードA）で個別関数を詰め、シェルスクリプト（モードB）で `staking.html` / `fundmanagement.html` のユーザーフローをまるごと通している。

### 2.4 Step 4: Aaveとのやりとりを書く

Aave V3 の WETH預入は **WETH Gateway経由** で ETHのまま（ラップ不要で）行える:

```solidity
// ETH → Aave → aWETH
wethGateway.depositETH{value: amount}(
    aavePool,          // pool
    address(this),     // onBehalfOf（aWETHの受取先）
    0                  // referralCode
);
```

引き出し時は **aWETHをapproveしてから** `withdrawETH`:

```solidity
// aWETH → ETH → this
aWETH.approve(address(wethGateway), amount);
wethGateway.withdrawETH(aavePool, amount, address(this));
```

DeInの [FundManagement.sol](solidity-contracts/src/FundManagement.sol) にも同じパターンが書かれている（`deployToAave` / `withdrawFromAave`）。

#### aWETHの独特な挙動に注意

`aWETH` は **リベース型（rebasing）トークン** で、保有者の `balanceOf` が時間経過で自動的に増える。これがAaveの利息。

```solidity
uint256 before = IERC20(aWETH).balanceOf(address(this));  // 1 ETH
vm.warp(block.timestamp + 30 days);
uint256 afterBal = IERC20(aWETH).balanceOf(address(this)); // 1.004 ETH（+0.4%/月程度）
```

> **注意**: リベースなので「1 ETH 預けた」→「1 ETH 引き出す」と **丸め誤差で1 wei 足りない** ことが発生する。[FundManagement.fork.t.sol の 243行目](solidity-contracts/test/fork/FundManagement.fork.t.sol#L243-L244) で `actualBalance * 99 / 100` と99%を引き出しているのはこの丸め誤差を避けるため。

### 2.5 Step 5: 時間を進めて利息を発生させる

フォーク環境の強みは **`vm.warp` と `vm.roll` で任意の時刻に飛べる** こと:

```solidity
vm.warp(block.timestamp + 30 days);
vm.roll(block.number + 216000);  // ~30日分のブロック
```

両方必要なのは、Aaveの内部計算が **`block.timestamp` ベース** の一方で、他のプロトコル（Compound等）は **`block.number` ベース** なので、片方だけ進めると一貫性が崩れる。`vm.warp` と `vm.roll` を必ずセットで呼ぶのがベストプラクティス。

anvil単独でやる場合は:

```bash
cast rpc evm_increaseTime 86401 --rpc-url $RPC_URL   # 24時間+1秒進める
cast rpc evm_mine --rpc-url $RPC_URL                 # ブロック採掘
```

[test-aave-fork.sh の 143行目](solidity-contracts/scripts/test-aave-fork.sh#L143-L144) がこのパターン。

### 2.6 Step 6: 他人のアカウントでなりすます（impersonate）

Aave ガバナンスの実行、他プロトコルの admin-only 関数の呼び出し、**特定ユーザーが持っている大量のtokenを借りて試験する** などに使う。

Foundry:

```solidity
vm.startPrank(0xSomeWhale);
IERC20(USDC).transfer(address(this), 1_000_000e6);
vm.stopPrank();
```

anvil:

```bash
cast rpc anvil_impersonateAccount 0xSomeWhale --rpc-url $RPC_URL
cast rpc anvil_setBalance 0xSomeWhale $(python3 -c "print(hex(50 * 10**18))") --rpc-url $RPC_URL
cast send --from 0xSomeWhale --unlocked ...
cast rpc anvil_stopImpersonatingAccount 0xSomeWhale --rpc-url $RPC_URL
```

DeInでは [test-aave-fork.sh の Part C](solidity-contracts/scripts/test-aave-fork.sh#L288-L423) で、**Lidoコントラクト自身をimpersonateして `finalize()` を呼ぶ** という荒技を使っている。本番では Lido のOracleしか `finalize` を呼べないが、フォークテストでは無理やり通してClaimフローを試している。

### 2.7 Step 7: 残高を好きなように差し込む（`vm.deal`）

特定アドレスにETHを生やす:

```solidity
vm.deal(address(treasury), 100 ether);
```

ERC20を生やしたいときはちょっと面倒で、**ストレージスロットに直接書き込む** 必要がある（`deal` チートコード、または `StdCheats`）。

### 2.8 一般手順のまとめ

```
[1] Aaveアドレスを定数化
      ↓
[2] RPCエンドポイント準備（ETH_RPC_URL）
      ↓
[3] モード選択（forge test --fork-url  /  anvil常駐）
      ↓
[4] setUp() で自作コントラクトをデプロイし、vm.deal で初期資金
      ↓
[5] deployToAave / withdrawFromAave 相当を呼ぶ
      ↓
[6] vm.warp で時間を進めて利息発生
      ↓
[7] 引き出して元本+利息を回収できるか assert
```

## 3. DeInプロジェクトでのフォークテスト構成の詳細

具体的にDeInがどう書いているかを、ファイル単位で見ていく。

### 3.1 `test/fork/FundManagement.fork.t.sol`

**位置づけ**: Foundryの単体テスト流儀（モードA）で、Aave / Lido / Curve 全部まとめて検証する。

主要なテスト:

| テスト名 | 検証内容 |
|---|---|
| `testFork_DeployToAave_Global` | Global Pool → Aave 預入で aWETH が受け取れる |
| `testFork_WithdrawFromAave` | Aaveから引き出し、Treasuryに資金が戻る |
| `testFork_AaveYieldAccumulation` | 30日経過でaWETHが自動増加（利息蓄積） |
| `testFork_DeployToLido` | Lidoに預入で stETH が受け取れる |
| `testFork_WithdrawFromLido` | CurveスワップでstETH → ETH（緊急用） |
| `testFork_EmergencyWithdrawAll` | Aave+Lido 全部引き出して Treasury に戻す |
| `testFork_LidoNativeWithdrawal_FullCycle` | Lido Native Withdrawalのリクエスト→finalize→Claim フル |

`MockTreasuryForFork` をTreasuryモックとして使い、`FundManagement` を本番相当で動かすという構成。Treasuryまで本物にすると設定が重くなるのでこの層だけモック化している。

### 3.2 `test/fork/FundManagement.fork.t.sol` の setUp の賢いところ

```solidity
try vm.envString("ETH_RPC_URL") returns (string memory) {
    // フォーク環境で実行
} catch {
    // 環境変数がない場合、このテストをスキップ可能にする
}
```

ETH_RPC_URLが設定されていない開発者でもテストスイートが壊れない。CIと個人マシンで挙動を切り替えられる。

### 3.3 `scripts/test-aave-fork.sh`

**位置づけ**: モードB。 anvil常駐 + cast叩きで、ユーザーフローをE2Eで再現する。

3パートに分かれている:

- **Part A**: `staking.html` のユーザー操作（stake / requestUnstake / executeWithdrawal）
- **Part B**: `fundmanagement.html` の運営操作（Aave/Lido 預入・引出し）
- **Part C**: Lido Native Withdrawal の完全再現（impersonateで finalize を強制）

フロントエンドが触る関数呼び出しとまったく同じ `cast send` を並べているので、**UIのE2Eテスト代わり** になっている。

### 3.4 デプロイスクリプト

```bash
forge script script/DeployStakingSystem.s.sol --rpc-url http://127.0.0.1:8545 --broadcast
```

[DeployStakingSystem.s.sol](solidity-contracts/script/DeployStakingSystem.s.sol) はStaking開発専用で、6コントラクト（DeInToken → TimelockController → DAOCore → Treasury → Staking → FundManagement）を一括デプロイする。フォーク環境でのデプロイ順序を固定化するためのもの。

### 3.5 DeIn のフォークテストで学んだ実務Tips

以下は実運用で詰まった点:

- **Chain IDは `31337`** で統一（Hardhat/Foundry/Anvilのデフォルト）
- **ウォレット競合**: MetaMaskで `localhost:8545` を開くと、ブラウザの接続履歴と anvil のnonceが食い違う事故が起きる。anvil起動のたびにMetaMask側で "Clear Activity Tab Data" が必要
- **gas limit を明示**: `--gas-limit 600000` を指定しないと、Aave系はガス推定が外れて revert するケースがある

## 4. 2026年4月18日、Aaveに何が起きたか — Kelp DAO rsETH事件

ここから後半。**Aaveのコード自体は破られていない** が、Aaveに預けられていた資金に深刻な影響が出た。

### 4.1 事件の結論を先に

- **2026年4月18日 18:52 UTC 前後**、Aave Guardian が Kelp DAO の rsETH 市場で異常を検知
- 攻撃者は **Kelp DAO の LayerZero ブリッジ** を侵害し、**116,500 rsETH（≈$292M、rsETH流通量の約18%）** を不正に発行・送金
- その偽 rsETH を **Aave V3 に担保として供給** し、**本物の WETH を借り入れて持ち逃げ**
- Aave に残った bad debt は **約 $196M**（ETHmainnet の rsETH-WETH ペアに集中）
- 24時間以内に Aave の TVL は **$22B → $15.4B（約$6.6B流出）** と急落、ETH/USDT/USDC pool は 100% 利用率に達し、一般預金者の引き出しが一時不能に

### 4.2 技術的な攻撃の仕組み — 「コードではなくインフラ構成」がやられた

攻撃の詳細は LayerZero の声明 と各メディアで明らかになってきている。

#### 攻撃ステップ

1. 攻撃者（LayerZero分析では**北朝鮮の Lazarus Group** と推定）が、**Kelp DAO の LayerZero メッセージ検証に使われていた2つのRPCノードを侵害**
2. これらのノード上で **稼働バイナリを差し替え**、LayerZero Verifier からの問い合わせにだけ **偽のトランザクション結果を返す** 改竄版を設置
3. 一方で他のモニタリングからの問い合わせには **正しい応答を返し続ける**（= 検知されない）
4. **DDoS攻撃** で Verifier を正常ノードから fail-over させ、侵害ノードに誘導
5. Verifier が「このクロスチェーンメッセージは正規」と誤認 → rsETH の不正mint命令が**正当なブリッジ通信として通る**
6. 偽 rsETH を手に入れた攻撃者が、流動性のある Aave に突っ込んで WETH を借りて逃走

#### なぜ成立したか — "1-of-1 Verifier" 構成

LayerZero の DVN（Decentralized Verifier Network）は、**複数のVerifierによる多数決（multi-sig的）** を許容する設計。Kelp DAO のブリッジは **1-of-1**（= LayerZero Labs だけが Verifier）で運用されていた。LayerZero 公式の integration checklist は **multi-verifier + 冗長性** を推奨していたが、Kelp はそれを採用していなかった（責任の所在はLayerZeroとKelpで互いに押し付け合いになっている）。

結果として、**「Verifier を1つ騙せれば通る」** という単一障害点が存在し、Lazarus のようなAPT攻撃に対して脆弱だった。

### 4.3 なぜAaveにbad debtが残ったのか — "composability"の宿命

ここが重要。**Aave のコードは一切 exploit されていない**。AaveのPool契約は「担保として供給された rsETH を受け取り、LTV（Loan-to-Value）に従って WETH を貸す」という当然の仕事をしただけ。

問題は **担保になった rsETH が、実際には何も裏付けがなくなった**（= 空気）こと。

```
[本来]                        [事件後]
rsETH = Kelp預託ETHの証文   →   rsETH = ただのトークン（裏付けなし）
                                 ^^^^ これがAaveに担保として入っている
```

Aaveのロジックからすると「担保は十分あるので貸す」だが、**担保の本当の価値が攻撃でゼロに近づいた**。清算しようにも流動性がなく、借入ポジションが返済されないまま貸し手側の損失に残る = **bad debt $196M**。

### 4.4 24時間でAaveから$6.6B流出した理由 — 連鎖パニックのメカニズム

bad debt が出た直後、Aave 預金者は「自分のETHも返ってこないかも」と考える。全員が同時に引き出しに走ると:

1. Pool の **ETH/USDT/USDC が100%利用率** に達する
2. 利用率100%では新規引き出しができない（借り手が返済しない限り）
3. 先に引き出した者勝ち、残された預金者は**最大2週間単位のunbonding期間に**トラップされる
4. 借り入れ側も金利急上昇で死亡、**$300M のさらなる借り入れスパイク**が発生（パニック借入）

つまり **bad debtは$196Mだが、流動性ショックでその30倍以上のTVLが流出** した。これが「composabilityリスク」の本質 — **プロトコル同士が相互依存しているので、ひとつが破れるとその余波が全体に波及する**。

### 4.5 Aaveの対応タイムライン

| 時刻（UTC） | 対応 |
|---|---|
| 4/18 ~18:00 | Kelp DAOブリッジへの異常検知 |
| 4/18 18:52 | Aave V3 の9つのネットワーク（Ethereum, Arbitrum, Base, Optimism, Prime等）で **rsETH / wrsETH 市場を凍結**（新規預入・借入停止） |
| 4/18 late | Aave V4 でも Protocol Security Council が凍結実施 |
| 4/19 02:28 | 予防的に WETH も凍結を検討（最終的には緊急引き出しを許可） |
| 4/19〜 | Stani Kulechov（Aave創業者）声明: "exploit is external, contracts not compromised" |
| 4/20〜 | ガバナンスフォーラムで bad debt の処理方針を議論中 |

Aave自体の凍結対応は技術的には**速かった**（検知→凍結まで1時間以内）。しかし **凍結しても既に借りられた WETH は返ってこない**。

## 5. DeInのステーキング設計への示唆

この事件は DeIn の設計に直接関わる。DeInのステーキングは:

```
ユーザーETH → Staking → Aave → aWETH → Treasury保管
```

というフローで、**Aaveそのものに依存している**。幸い、DeInは rsETH のような LRT を扱っていないが、Aave側で同様の事件が再発すれば **DeInの預け入れ ETH も引き出せなくなるリスク** がある。

### 5.1 DeInが被りうるリスクを分解する

| リスクカテゴリ | 具体例 | DeInへの影響 |
|---|---|---|
| **① Aaveコントラクト自体のexploit** | Aave V3 のロジックにバグ（過去事例: 他レンディングプロトコルで散発） | aWETH の価値が毀損、引き出し不能 |
| **② Aaveに紐づく担保資産のexploit**（**今回これ**） | Kelp rsETH のような別アセットの bad debt で Aave pool が枯渇 | WETH pool が 100% 利用率 → 引き出しロック |
| **③ オラクル攻撃** | Aave のpriceフィードが狂う | 予期せぬ清算が発生、Treasury保有 aWETH が減る |
| **④ ガバナンス攻撃** | Aave DAO が悪意あるアップグレードを通す | aWETHの性質が変わる、最悪焼却される |
| **⑤ 流動性ショック** | パニック引き出しで pool が枯渇 | 一時的に Treasury が ETH を取り戻せない |

今回の事件は **② と ⑤** の合せ技だった。

### 5.2 特に ⑤（流動性ショック）がDeInに与える影響は重い

DeInのユースケースを考えると、**「ステーカーのアンステークや保険金支払い」は地震発生後に集中する**可能性がある。そのタイミングでAaveのpoolが枯渇していたら:

- **ユーザーはアンステーク不能** → ユーザー信頼の崩壊
- **保険金支払い原資をTreasuryから引き出せない** → 受益者への支払い遅延
- **そもそも「地震が起こると資金運用益が減る」構造なのに、さらに二重に打撃**

という詰みパターンがあり得る。DeInは「地震発生時の資金返還」こそ最も信頼性が求められるシステムなので、**Aaveが一時凍結されたらDeInも一時凍結する** という挙動はビジネス的に許容できない。

### 5.3 設計を再考すべき具体的ポイント

**結論から言うと、以下の方向性で設計を見直したい:**

#### ① 単一プロトコル依存の回避（diversification）

現状 DeIn のステーキング ETH は **全額 Aave V3**。これを:

- Aave V3 に X%
- Lido (stETH) に Y%
- Compound v3 / Morpho / Spark などに Z%

のように **複数のプロトコルに分散** すべき。コード面では、`FundManagement.sol` で既に Aave + Lido の2プロトコル対応になっているので、**Stakingのデフォルト預入先も分散**に踏み込むのが自然な次ステップ。

#### ② 「即時引き出し枠」を常に確保する

現状、預かった ETH は全額 Aave に流す設計（利回りを最大化するため）。しかし **直近数日分のアンステーク需要相当（例: 全ステーク額の5%）は Treasury にETHのまま保持** すべき。Aaveがロックされても最低限の引き出しは応じられる状態にする。

#### ③ LRT（Liquid Restaking Token）には近づかない

Kelpの rsETH、Ether.fiの eETH、Renzoの ezETH のような **LRT（Liquid Restaking Token）** は、**バックエンドがクロスチェーンブリッジに依存**しているケースが多く、まさに今回のようなブリッジexploitに対して脆弱。

**保険金の原資**というクリティカル資金の運用先としては、**EigenLayer再ステーキング系の利回り上乗せは追わない** という方針を明文化したい。利回りを狙って攻撃リスクを取り込むのは本末転倒。

#### ④ オラクル多重化・凍結機構

Aave自体が凍結されたら、DeInも即座に **Aaveから引き出す／新規預入を止める** 緊急モードを発動する設計にする。`FundManagement.sol` には既に `emergencyMode` と `emergencyWithdrawAll` があるので、これを **外部プロトコルの凍結イベント連動**で自動発火できるようにすると良い。

ただし「Aaveが凍結されているときに引き出そうとする」= **枯渇pool から引き出せないので結局失敗**という矛盾もある。現実には「事前の早期警戒→ポジション削減」が主、「事後の緊急撤退」は補助、という二層運用になる。

#### ⑤ bad debtに対する明示的な損失分担ルール

万一 DeIn の Treasury が Aave から ETH を全額回収できなかった場合、**誰が損を被るのか**を設計しておく必要がある:

- **オプションA**: 保険金支払いを優先し、ステーカーが損失を被る（ステーカーの元本減額）
- **オプションB**: ステーカーの元本を優先し、保険金減額（被保険者が被る）
- **オプションC**: DAO Treasury の Global Pool が補填（DAO全体で吸収）

現状の `Staking.sol` には `stakerHaircutFactor` という仕組みがあり、**保険金支払い時にステーカーがhaircutを食らう** 設計になっている。これを **「外部プロトコル損失時にも適用する」** のか、**「外部プロトコル損失は別途Global Poolで補填する」** のか、ホワイトペーパーで明示すべき。

### 5.4 フォークテストに追加したいシナリオ

今回のような事件に耐える設計かを検証するため、以下のフォークテストを追加したい:

| テスト | 検証内容 |
|---|---|
| `testFork_AavePoolAt100PctUtilization` | Aave poolの利用率を強制的に100%にし、Treasury が引き出し失敗を適切にハンドルするか |
| `testFork_AaveBadDebtScenario` | Aave Poolに架空のbad debtを差し込み、aWETHの実効価値下落を再現 → Treasury残高評価が正しく下がるか |
| `testFork_EmergencyWithdrawalUnderStress` | パニック状況でも `emergencyWithdrawAll` が完走するか（ガス限界・スリッページ考慮） |
| `testFork_MultiProtocolDiversification` | 預け入れ先を複数プロトコルに分散したとき、1つが凍結されても他から引き出せるか |

既存の [FundManagement.fork.t.sol](solidity-contracts/test/fork/FundManagement.fork.t.sol) は「正常系の利息蓄積・引き出し」がメインだが、**「異常系」をフォーク上で再現するテストが不足**している。これは今回の事件を教訓に最優先で埋めたい。

## 6. Q&A: よくある疑問

### Q1: 「mainnet フォーク」はmainnetを改ざんするの？

**しない**。フォーク元のmainnet状態はRPC経由で**読み取り専用**でコピーされるだけ。ローカルで改ざんしてもそれがmainnetに書き戻されることはない。完全にローカルサンドボックス。

### Q2: 「AaveのコードがexploitされていないならDeInは関係なくない？」

**関係大アリ**。コードがクリーンでも、**pool の流動性が枯渇すれば引き出せない**。DeInはAaveを「安全な利息原資」として使っていたが、**Aaveは "コード" + "預かっている資産全体" の総合システム**であり、後者が腐るとDeInも道連れになる。これはEthereumエコシステム全体が抱える composability risk。

### Q3: 「Lidoなら安全？」

Lido (stETH) はLRTと違ってEthereumネイティブのバリデーターに直接ステーキングしているので、**クロスチェーンブリッジのリスクは小さい**。ただし Lidoにも過去 Oracle問題や stETH の depeg（ペグ外れ）事件はあったので、過信は禁物。**「どのプロトコルも単独では信頼しない、組み合わせで耐える」** が健全。

### Q4: なぜEthereumには "sudo で全員のAave引き出しをキャンセル" みたいな管理者権限がないのか

まさに**それがブロックチェーンの価値**。中央集権的な巻き戻しができない代わりに、誰も勝手に資産を没収できない。Aaveの凍結権限（Guardian）は「新規預入・借入を止める」までで、**既に借りられた分を巻き戻す権限は持たない**。この設計が事件時に脆さを露呈するが、平時の信頼性の源でもあるというトレードオフ。

### Q5: 「フォークテストはmainnetに課金される？」

**されない**。フォーク元mainnetにはただの**read-only RPCコール**が飛ぶだけ。ガス代もかからない。ローカル側のanvilはガスを**0 または仮想的に**扱う。ただしRPC provider（Alchemy等）のリクエスト数制限には引っかかりうる。

## 7. まとめ

### 本日学んだこと（前半）

- **フォークテスト**は「mainnet状態をローカルに遅延コピーしてテストする」手法で、DeFi連携コントラクトのテストには必須
- Foundryには `forge test --fork-url` という組み込みフォーク機能と、`anvil --fork-url` + スクリプトでの常駐方式の**2流派**がある
- **DeIn** は両方使っていて、前者で単体検証、後者でユーザーフローE2E を回している
- Aave V3 とのやりとりは **WETH Gateway** 経由で ETHのまま預けられる、aWETHはrebase型なので丸め誤差注意
- **時間進行**（vm.warp）と **impersonate**（vm.startPrank / anvil_impersonateAccount）はフォークテストの2大武器

### 2026年4月18日事件の教訓（後半）

- **AaveのコードはexploitされていないがAaveエコシステム全体が打撃を受けた** のが最大のポイント
- 攻撃の本体は **Kelp DAO の LayerZero 1-of-1 Verifier構成**。インフラ構成の単一障害点が突かれた（Lazarus Group推定）
- **$292M** が盗まれ、そのうち **$196M** が Aave に bad debt として残存
- 24時間で Aave TVL が **$22B → $15.4B** に急落、**pool 100% 利用率** で一般預金者が引き出し不能に
- これは **composability risk** の典型。プロトコル同士が相互依存する以上、ひとつの事件が波及する

### DeInのステーキング設計の次のアクション

1. **預入先の分散** — Aave単一からAave+Lido+他プロトコルに
2. **即時引き出し枠の確保** — 一部ETHを Treasury に現金のまま保持
3. **LRT（Kelp/EtherFi/Renzo系）は採用しない** ことを明文化
4. **異常系フォークテスト**を追加（pool 100%利用率、bad debt 混入、緊急撤退）
5. **外部プロトコル損失時の損失分担ルール**をホワイトペーパーに明記

今回の事件は、DeInが「なぜリスクをオンチェーンで可視化して分散する保険プロトコルなのか」という立脚点にもう一度立ち戻らせるもの。**自分たちもその composability の網の一部であり、何に依存しているかを常に監視する責任がある**ということを肝に銘じたい。

---

## 参考文献

- [rsETH incident — 2026-04-18 - Aave Governance](https://governance.aave.com/t/rseth-incident-2026-04-18/24481)
- [KelpDAO Incident Statement - LayerZero](https://layerzero.network/blog/kelpdao-incident-statement)
- [2026's biggest crypto exploit: $292 million gets drained from Kelp DAO - CoinDesk](https://www.coindesk.com/tech/2026/04/19/2026-s-biggest-crypto-exploit-kelp-dao-hit-for-usd292-million-with-wrapped-ether-stranded-across-20-chains)
- [LayerZero blames Kelp's setup for $290 million exploit - CoinDesk](https://www.coindesk.com/tech/2026/04/20/layerzero-blames-kelp-s-setup-for-usd290-million-exploit-attributes-it-to-north-korea-s-lazarus)
- [Aave records $6 billion TVL drop as Kelp hack exposes structural risk - CoinDesk](https://www.coindesk.com/tech/2026/04/19/aave-records-usd6-billion-tvl-drop-as-kelp-hack-exposes-structural-risk-at-defi-lender)
- [A $300 million borrowing spike on Aave signals liquidity crunch - CoinDesk](https://www.coindesk.com/markets/2026/04/20/a-usd300m-borrowing-spike-on-aave-signals-liquidity-crunch-after-exploit)
- [How a Single LayerZero DVN Compromise Drained $292M from KelpDAO - Blockaid](https://www.blockaid.io/blog/how-a-single-layerzero-dvn-compromise-drained-292m-from-kelpdao)
- [Aave V3 公式アドレス一覧](https://aave.com/docs/resources/addresses)
- [Foundry Book - Fork Testing](https://book.getfoundry.sh/forge/fork-testing)
