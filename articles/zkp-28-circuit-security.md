---
title: "[ゼロ知識証明入門シリーズ 28/33] 回路セキュリティ — Under-Constrained バグとの戦い"
emoji: "🛡️"
type: "tech"
topics:
  - zkp
  - security
  - underconstrained
  - audit
  - formal
published: false
---

**日付**: 2026年4月22日
**学習内容**: ZKP 実装で最も致命的なバグが **Under-Constrained バグ**（制約不足）。検証は通るのに、**存在しない witness を通してしまう**。これまでに Tornado Cash, Zcash, Aztec などで発見・修正された脆弱性の多くがこのパターン。本記事では **(1) ZKP 特有のバグカテゴリ**、**(2) Under-Constrained とは何か**、**(3) 実例（unchecked outputs, wrong selector）**、**(4) Over-Constrained（別の致命バグ）**、**(5) 形式検証ツール (Vanguard, Picus, ecne)**、**(6) ベストプラクティス**、**(7) ZK 監査プロセス** を扱う。

## 0. 本記事の位置づけ

ZKP の「数学的健全性」は厳密だが、**実装ミスで全部が崩れる**。しかも普通のプログラムと違い、**Under-Constrained バグはテストで発見しにくい**:

- 正常入力では動く
- 悪意ある Prover だけが異常を引き起こす

Ethereum ブリッジや Zcash クラスの損害が数十億ドル規模になることを考えると、**回路のセキュリティ**は ZKP 実装の最重要テーマ。

構成:

- **第1章**: ZKP バグの特殊性
- **第2章**: Under-Constrained バグ
- **第3章**: 典型的パターンと実例
- **第4章**: Over-Constrained バグ
- **第5章**: 形式検証ツール
- **第6章**: ベストプラクティス
- **第7章**: 監査プロセス
- **第8章**: Q&A とまとめ

## 1. ZKP バグの特殊性

### 1.1 通常のバグとの違い

| 項目 | 通常のプログラム | ZKP 回路 |
|---|---|---|
| テスト可能性 | 動作を観察できる | 正当性を観察しにくい |
| 攻撃者の役割 | 通常は入力 | Prover 自身が攻撃者 |
| バグの検出 | クラッシュ・異常値 | 検証が「通ってしまう」 |
| 影響範囲 | 特定の入力 | すべての証明 |

### 1.2 Prover は悪意ありと仮定

ZKP の設計では**Prover を信頼しない**のが前提。Prover は任意の signal 値を選べるので、**制約で witness を一意に固定しなければならない**。

### 1.3 「証明が通る」=「回路が正しい」ではない

普通のプログラムで「実行結果が期待通り」なら正しい。ZKP では **「Prover が提示した証明が検証された」からといって、Prover の計算が正しい保証はない**（制約が足りていなければ）。

## 2. Under-Constrained バグ

### 2.1 定義

**Under-Constrained**: 制約が**本来の計算を一意に決めるのに不足**している状態。Prover は計算をズルできる。

### 2.2 最も単純な例

「$y = x^2$」を示す回路:

```circom
template Square() {
    signal input x;
    signal output y;

    y <-- x * x;  // 代入のみ (BUG)
}
```

`<--` は**witness 代入のみで制約がない**。Prover は $y$ に任意の値を入れられる。

正しくは:

```circom
y <== x * x;  // 代入 + 制約
```

### 2.3 もう少し巧妙な例

「$b$ はブール値」:

```circom
signal b;
// b * (b - 1) === 0 の制約なし (BUG)
```

この制約がないと Prover は $b = 7$ でも通せる。

### 2.4 出力の未制約

```circom
template MultiOutput() {
    signal input x;
    signal output a;
    signal output b;

    a <== x * 2;
    // b は一切制約されていない (BUG)
}
```

Prover は `b` に任意の値を入れて通せる。**出力が未制約**は ZKP で最も多いバグ。

## 3. 典型的パターンと実例

### 3.1 Pattern 1: `<--` の乱用

```circom
signal q, r;
q <-- x \ y;
r <-- x % y;
// ↑ これだけでは「q, r が本当に x = q*y + r を満たす」保証なし
```

正しい形:

```circom
q <-- x \ y;
r <-- x % y;
x === q * y + r;      // 追加制約
r * (r - (y - 1)) === 0;  // 範囲制約（簡略化）
```

### 3.2 Pattern 2: 比較の不完全

```circom
template LessThan() {
    signal input a;
    signal input b;
    signal output out;

    out <-- a < b ? 1 : 0;  // ビット分解してない (BUG)
}
```

正しくは circomlib の `LessThan` を使う（ビット分解 + キャリー処理）。

### 3.3 Pattern 3: Range check 忘れ

```circom
signal amount;
amount <== input;  // amount が [0, 2^256) の範囲か？
```

$\mathbb{F}_p$ での負数は $p - n$ として扱われる。範囲制約がないと Prover は**負の残高**や**オーバーフロー値**を使える。

### 3.4 Pattern 4: Zero division

```circom
signal inv;
inv <-- 1 / x;  // x = 0 のとき？
```

$x = 0$ でも Prover が inv に任意の値を入れられる。正しくは:

```circom
signal inv;
signal nonZero;

inv <-- x != 0 ? 1 / x : 0;
x * inv === nonZero;   // nonZero が 1 if x != 0
nonZero * (nonZero - 1) === 0;
```

### 3.5 実例1: Tornado Cash の初期バージョン

2019 年の初期版で、**Merkle tree の leaf 位置**が under-constrained。攻撃者が同じ leaf を複数回引き出せる可能性。commit 前に修正。

### 3.6 実例2: Semaphore v1

Iden3 の Semaphore v1 で nullifier の生成に under-constrained。nullifier が一意にならず、**同じ voter が複数回投票できる**バグ。v2 で修正。

### 3.7 実例3: ZkSync の早期実装

ZkSync の早期実装で、**balance update の constraint が不完全**。Paradigm の監査で発見・修正。

### 3.8 実例4: Circom 2.0 の Num2Bits

2022 年に発見された **Num2Bits** (circomlib) のバグ。入力 $x$ が $2^n$ 以上でも通ってしまう場合があった。`Num2Bits_strict` に置き換え推奨。

## 4. Over-Constrained バグ

### 4.1 定義

**Over-Constrained**: 制約が**多すぎて**、**正しい計算が通らない**。

### 4.2 影響

- Prover が正当な証明を作れない
- 特定の入力でだけ失敗
- DoS 的な影響

### 4.3 例: 意図しない等式

```circom
template Buggy() {
    signal input a;
    signal input b;
    signal output c;

    c <== a + b;
    a === 0;  // BUG: 不必要な制約
}
```

`a = 0` しか通らなくなる。

### 4.4 Under-Constrained よりはマシだが

Under-Constrained は**健全性を破る**（致命的）。Over-Constrained は**完全性を破る**（機能不全）。どちらもダメだが、前者の方が危険。

## 5. 形式検証ツール

### 5.1 Vanguard (USC + Trail of Bits, 2023)

Circom 回路の静的解析ツール:

- **Circuit Dependency Graph (CDG)** を構築
- signal 間の依存関係を追跡
- 未制約の signal を警告

使い方:

```bash
python3 vanguard.py my_circuit.circom
```

### 5.2 Picus (UT Austin, 2022)

**SMT ソルバーベース**の formal verifier:

- 回路を SMT 制約に変換
- Z3 などで「すべての signal が一意に決まるか」を判定
- Under-Constrained なら反例を返す

強力だが、大規模回路では時間がかかる。

### 5.3 ecne (0xParc, 2022)

Graph-based の早期検出ツール。Vanguard の先行研究。

### 5.4 ZKAP (Quantstamp)

商用の ZKP 監査フレームワーク。複数ツールを統合。

### 5.5 コルベス (Veridise)

Coq や Lean ベースの形式証明。最も厳密だが**証明を書くコストが高い**。

### 5.6 ツール選択

- **軽量な pre-check**: Vanguard, ecne
- **深い検証**: Picus
- **高い信頼性必須**: Coq / Lean（研究機関レベル）

## 6. ベストプラクティス

### 6.1 `<--` の使用を最小化

`<--` は witness 計算のためだけに使い、**必ず `===` で制約を追加**。

### 6.2 出力に必ず制約

すべての output signal は、入力から一意に決まる**制約を書く**。

### 6.3 範囲検証を徹底

整数的な計算には range check を忘れず:

```circom
include "node_modules/circomlib/circuits/bitify.circom";

component rangeCheck = Num2Bits(32);
rangeCheck.in <== x;
// これで x < 2^32 が保証
```

### 6.4 既知のライブラリを使う

circomlib の最新版を使う。独自実装より**既存の audit 済みコード**を優先。

### 6.5 複数実装でのクロスチェック

Circom での実装と、**Rust (arkworks)** での参照実装を比較。両方で同じ witness / 証明が出れば高確率で正しい。

### 6.6 Fuzz Testing

ランダム入力で**Prover が通せる異常ケース**を探索:

```javascript
for (let i = 0; i < 10000; i++) {
    const input = randomInput();
    try {
        const witness = calculateWitness(input);
        if (!isValid(input) && isConstraintSatisfied(witness)) {
            console.log("Under-Constrained bug!", input);
        }
    } catch (e) {}
}
```

### 6.7 Property-Based Testing

「任意の入力 $x$ に対して ...」という性質をテスト:

```javascript
test.prop([fc.integer()], (x) => {
    const witness = calculateWitness({ x });
    expect(witness.y).toEqual(x * x);
});
```

## 7. 監査プロセス

### 7.1 監査の一般的フロー

1. **仕様書レビュー**: 何を証明したいか、数学的要件
2. **コード review**: 制約の完全性チェック
3. **形式検証ツール**: Vanguard/Picus
4. **Manual review**: エキスパートによる手動確認
5. **Cross-check**: 別実装との比較
6. **Fuzzing**: ランダム入力でバグ探索
7. **Report**: 発見バグと修正推奨

### 7.2 監査会社

- **Trail of Bits**
- **ChainSafe**
- **Veridise**
- **Quantstamp**
- **ZK Security**

いずれも ZKP 特化チームあり。

### 7.3 バグ報奨金

多くの ZKP プロジェクトで**バグ報奨金（bug bounty）** が設定されている:

- **Immunefi**: Aztec, zkSync, StarkNet など
- **Code4rena**: 競技的監査

Under-Constrained バグは**最高額カテゴリ** (Critical) で、数十万〜数百万ドル。

### 7.4 継続的監査

- コード更新のたびに再監査
- Formal verification を CI に組み込む

## 8. Q&A

### Q1: テストでは何ができる？

「正常ケースで witness が期待通り」「正常ケースで constraint が満たされる」はテスト可能。**Under-Constrained の検出**は悪意 prover のシミュレーションが必要で、普通の unit test では困難。

### Q2: Under-Constrained のバグは Groth16 特有？

**SNARK 全般で起こる**。R1CS / Plonkish / AIR どこでも制約の設計ミスで発生。PLONK の wiring constraint でも同様。

### Q3: Halo2 での対策は？

Halo2 は Rust ベースで**型システム**が厳格。signal の使い忘れをコンパイル時に検出。ただし数学的正当性は別問題なので、formal verification 推奨。

### Q4: STARK は Circom のような DSL がある？

- **Cairo** (StarkNet): Turing-complete に近い、専用
- **Noir** (Aztec): Rust 風
- **Miden** (Polygon): アセンブリ風

各言語固有の pitfall がある。

### Q5: 回路の複雑さと安全性の関係？

**複雑 = バグの温床**。zkEVM のような巨大回路ほど監査が難しい。モジュラー設計と formal verification が必須。

### Q6: 攻撃されたらどうなる？

Under-Constrained バグが本番で exploit されると:

- 不正な送金・引き出し
- Nullifier 再利用
- 無制限な mint
- Rollup state の改竄

**事後の回復は困難**（ブロックチェーンは不可逆）。したがって**事前の検証**が最重要。

## 9. まとめ

### 本記事の要点

1. **Under-Constrained バグ**: 制約不足、Prover が計算をズルできる
2. 典型パターン: `<--` 乱用、範囲検証忘れ、ゼロ除算、出力未制約
3. **Over-Constrained**: 制約過剰、完全性を破る
4. **形式検証ツール**: Vanguard, Picus, ecne
5. **ベストプラクティス**: `<==` 優先、範囲検証、既知ライブラリ、fuzz test
6. **監査プロセス**: 仕様レビュー → tool → manual → fuzz → report
7. 監査会社と bug bounty の活用

### 次の記事（Article 29）へ

次の記事から **第7部: 応用と実例**。最初は **Zcash** を扱う。世界初の大規模プライバシーコインで、Sprout → Sapling → Orchard と 3 世代の SNARK 進化を経てきた。各世代の仕組みを追う。

### 3行サマリ

- **Under-Constrained = 制約不足で Prover が計算を捏造できる致命的バグ**
- **`<--` の乱用、範囲検証忘れ、ゼロ除算**が典型
- **Vanguard / Picus など formal 検証ツール** + 監査が必須

---

## 参考文献

- Jon Stephens et al. *Practical Security Analysis of Zero-Knowledge Proof Circuits.* USENIX Security 2024.
- Trail of Bits. *ZKP Audit Guidelines.* 2023.
- Veridise. *Vanguard Static Analysis.* 2023.
- Tornado Cash. *Tornado.cash Security Audit.* 2019.
- ZKP MOOC Lecture 15 (UC Berkeley, 2023).
