---
title: "[完全準同型暗号入門シリーズ 14/15] 実装 — SEAL・OpenFHE・TFHE-rs・Concrete"
emoji: "🛠️"
type: "tech"
topics:
  - fhe
  - seal
  - openfhe
  - tfhers
  - concrete
published: false
---

**日付**: 2026年4月24日
**学習内容**: 本記事は FHE の **実装編**。これまでの理論を実際に動かすための **6 つの主要ライブラリ** を比較し、それぞれでの実装例を紹介する。具体的には **(1) Microsoft SEAL (C++/C#)**、**(2) OpenFHE (C++, Python binding)**、**(3) HElib (C++)**、**(4) TFHE-rs (Rust)**、**(5) Concrete (Python)**、**(6) Lattigo (Go)** を取り上げる。さらに **(7) 環境構築**、**(8) BFV による秘匿集計**、**(9) CKKS による暗号化 ML 推論**、**(10) TFHE による Boolean 回路** の動くコード例、**(11) ベンチマーク比較**、**(12) デバッグのコツ** を扱う。理論だけでなく「**動かせる**」レベルまで踏み込む。

## 0. 本記事の位置づけ

Article 11-13 で BGV/BFV/CKKS/TFHE を学んだ。しかし理論だけでは実装できない。本記事は:

- どのライブラリを選ぶか
- どう環境構築するか
- 典型的なコードはどう書くか
- どこで詰まるか

を実際的に扱う。

構成:

- **第1章**: 主要ライブラリの比較
- **第2章**: Microsoft SEAL
- **第3章**: OpenFHE
- **第4章**: TFHE-rs
- **第5章**: Concrete
- **第6章**: HElib, Lattigo
- **第7章**: BFV 秘匿集計の実装
- **第8章**: CKKS ML 推論の実装
- **第9章**: TFHE Boolean 回路
- **第10章**: ベンチマーク
- **第11章**: デバッグ
- **第12章**: Q&A とまとめ

## 1. 主要ライブラリの比較

### 1.1 概観

| ライブラリ | 言語 | スキーム | 開発者 | 特徴 |
|---|---|---|---|---|
| **SEAL** | C++/C# | BFV, CKKS, BGV | Microsoft | 業界事実上の標準、API 綺麗 |
| **OpenFHE** | C++ (+Py) | 全部 (BGV, BFV, CKKS, TFHE(DM/CGGI)) | Duality 中心のコンソーシアム | 最も包括的、学術向き |
| **HElib** | C++ | BGV, CKKS | IBM | 古株、BGV の bootstrap 最速 |
| **TFHE-rs** | Rust | TFHE | Zama | TFHE のリファレンス実装 |
| **Concrete** | Python | TFHE | Zama | Python → FHE 自動コンパイル |
| **Lattigo** | Go | BFV, CKKS, BGV | EPFL / Tune Insight | Go 製、並列処理が得意 |

### 1.2 選び方の指針

- **汎用 BFV/CKKS 学習** → **SEAL** (ドキュメント豊富、サンプル多数)
- **最新研究** → **OpenFHE** (全スキーム入り)
- **TFHE 専用** → **TFHE-rs** or **Concrete**
- **Python で手軽に** → **Pyfhel** (SEAL ラッパー) or **Concrete**
- **Go 環境** → **Lattigo**
- **IBM 互換** → **HElib**

### 1.3 ライセンス

- SEAL, OpenFHE, HElib, Lattigo: **Apache 2.0 / BSD 系** (商用 OK)
- TFHE-rs, Concrete: **BSD-3-Clause-Clear** (商用 OK、2023 以降)

いずれも **商用利用可能**。

## 2. Microsoft SEAL

### 2.1 インストール

```bash
git clone https://github.com/microsoft/SEAL.git
cd SEAL
cmake -S . -B build
cmake --build build
sudo cmake --install build
```

Python バインディングは **Pyfhel** や **SEAL-Python**。

### 2.2 API の基本構造

```cpp
#include "seal/seal.h"
using namespace seal;

int main() {
    // パラメータ設定
    EncryptionParameters parms(scheme_type::bfv);
    parms.set_poly_modulus_degree(8192);
    parms.set_coeff_modulus(CoeffModulus::BFVDefault(8192));
    parms.set_plain_modulus(1024);
    
    // コンテキスト作成
    SEALContext context(parms);
    
    // 鍵生成
    KeyGenerator keygen(context);
    auto secret_key = keygen.secret_key();
    PublicKey public_key;
    keygen.create_public_key(public_key);
    RelinKeys relin_keys;
    keygen.create_relin_keys(relin_keys);
    
    // 操作オブジェクト
    Encryptor encryptor(context, public_key);
    Evaluator evaluator(context);
    Decryptor decryptor(context, secret_key);
    BatchEncoder encoder(context);
    
    // 使用...
}
```

### 2.3 典型的なワークフロー

1. **Parameters → Context → KeyGen**
2. **Encoder** で平文 → Plaintext
3. **Encryptor** で Plaintext → Ciphertext
4. **Evaluator** で準同型演算（add, multiply, rotate）
5. **Decryptor + Encoder** で復号 → 平文

### 2.4 サンプルコード

SEAL リポジトリの `native/examples/` に:

- `1_bfv_basics.cpp`
- `2_encoders.cpp`
- `3_levels.cpp`
- `4_ckks_basics.cpp`
- `5_rotation.cpp`
- `6_serialization.cpp`
- `7_performance.cpp`

順に読むと基礎が分かる。

## 3. OpenFHE

### 3.1 特徴

- **全スキーム**実装（BGV, BFV, CKKS, DM/CGGI/FHEW/TFHE）
- **CKKS bootstrap** が実装されている
- **Python bindings** が公式サポート

### 3.2 インストール

```bash
# C++
git clone https://github.com/openfheorg/openfhe-development.git
cd openfhe-development
mkdir build && cd build
cmake ..
make -j
sudo make install

# Python
pip install openfhe
```

### 3.3 Python での CKKS 例

```python
from openfhe import *

# パラメータ
mult_depth = 5
scale_mod_size = 50
first_mod_size = 60

parameters = CCParamsCKKSRNS()
parameters.SetMultiplicativeDepth(mult_depth)
parameters.SetScalingModSize(scale_mod_size)
parameters.SetFirstModSize(first_mod_size)

cc = GenCryptoContext(parameters)
cc.Enable(PKE)
cc.Enable(LEVELEDSHE)
cc.Enable(ADVANCEDSHE)

# 鍵生成
keys = cc.KeyGen()
cc.EvalMultKeyGen(keys.secretKey)

# 暗号化
input_values = [1.1, 2.2, 3.3, 4.4]
plaintext = cc.MakeCKKSPackedPlaintext(input_values)
ciphertext = cc.Encrypt(keys.publicKey, plaintext)

# 準同型乗算
ct_squared = cc.EvalMult(ciphertext, ciphertext)

# 復号
decrypted = cc.Decrypt(ct_squared, keys.secretKey)
decrypted.SetLength(len(input_values))
print(decrypted)
# ≈ [1.21, 4.84, 10.89, 19.36]
```

### 3.4 Bootstrap の使用

```python
cc.Enable(FHE)
cc.EvalBootstrapSetup([3, 3])
cc.EvalBootstrapKeyGen(keys.secretKey, batch_size)

# 重い計算...
ct_bootstrapped = cc.EvalBootstrap(ciphertext)
```

## 4. TFHE-rs

### 4.1 インストール

```bash
# Cargo.toml に追加
[dependencies]
tfhe = { version = "0.8", features = ["boolean", "shortint", "integer", "x86_64-unix"] }
```

### 4.2 基本の例

```rust
use tfhe::prelude::*;
use tfhe::{ConfigBuilder, FheUint32, generate_keys, set_server_key};

fn main() {
    let config = ConfigBuilder::default().build();
    let (client_key, server_key) = generate_keys(config);
    set_server_key(server_key);
    
    // 暗号化
    let a = FheUint32::encrypt(1234u32, &client_key);
    let b = FheUint32::encrypt(5678u32, &client_key);
    
    // 演算
    let c = &a + &b;
    let d = &a * &b;
    let e = a.gt(&b);   // a > b
    
    // 復号
    let c_dec: u32 = c.decrypt(&client_key);
    let d_dec: u32 = d.decrypt(&client_key);
    let e_dec: bool = e.decrypt(&client_key);
    
    println!("a + b = {}", c_dec);
    println!("a * b = {}", d_dec);
    println!("a > b = {}", e_dec);
}
```

### 4.3 Boolean 回路

```rust
use tfhe::boolean::prelude::*;

fn main() {
    let (client_key, server_key) = gen_keys();
    
    let a = client_key.encrypt(true);
    let b = client_key.encrypt(false);
    
    // 論理ゲート
    let and_res = server_key.and(&a, &b);
    let or_res  = server_key.or(&a, &b);
    let xor_res = server_key.xor(&a, &b);
    let nand_res = server_key.nand(&a, &b);
    
    println!("true AND false = {}", client_key.decrypt(&and_res));
}
```

## 5. Concrete

### 5.1 インストール

```bash
pip install concrete-python
```

### 5.2 基本の例

```python
from concrete import fhe

# 関数をデコレータで指定
@fhe.compiler({"x": "encrypted"})
def square(x):
    return x ** 2

# コンパイル（代表的な入力で）
inputset = [(i,) for i in range(0, 16)]
circuit = square.compile(inputset)

# 実行
result = circuit.encrypt_run_decrypt(7)
print(result)  # 49
```

### 5.3 複雑な関数: ReLU

```python
@fhe.compiler({"x": "encrypted"})
def relu(x):
    return fhe.if_then_else(x > 0, x, 0)

circuit = relu.compile([(i,) for i in range(-10, 11)])
print(circuit.encrypt_run_decrypt(-3))  # 0
print(circuit.encrypt_run_decrypt(5))   # 5
```

### 5.4 ML モデル推論 (Concrete ML)

```python
from concrete.ml.sklearn import LogisticRegression
from sklearn.datasets import make_classification

# 通常の sklearn 風
X, y = make_classification(n_samples=100, n_features=5)
model = LogisticRegression(n_bits=4)
model.fit(X, y)

# FHE にコンパイル
model.compile(X)

# 暗号化推論
y_pred_fhe = model.predict(X[:5], fhe="execute")
print(y_pred_fhe)
```

## 6. HElib, Lattigo

### 6.1 HElib

IBM 謹製。**BGV の bootstrap** が最速。

```cpp
#include <helib/helib.h>

Context context = ContextBuilder<BGV>()
    .m(32768)
    .p(17)
    .r(1)
    .bits(500)
    .c(2)
    .build();

SecKey secret_key(context);
secret_key.GenSecKey();
addSome1DMatrices(secret_key);
PubKey& public_key = secret_key;

// 暗号化・復号・演算...
```

### 6.2 Lattigo

Go 製。並列処理・マルチコアに強い。

```go
import "github.com/tuneinsight/lattigo/v6/schemes/ckks"

params, _ := ckks.NewParametersFromLiteral(ckks.PN14QP438)
kgen := ckks.NewKeyGenerator(params)
sk, pk := kgen.GenKeyPair()
enc := ckks.NewEncryptor(params, pk)
dec := ckks.NewDecryptor(params, sk)
eval := ckks.NewEvaluator(params, nil)

// 暗号化・演算・復号...
```

## 7. BFV 秘匿集計の実装

### 7.1 ユースケース

**複数人の給与の平均を、個々の額を漏らさず計算する**。

### 7.2 SEAL での実装

```cpp
#include "seal/seal.h"
using namespace std;
using namespace seal;

int main() {
    EncryptionParameters parms(scheme_type::bfv);
    parms.set_poly_modulus_degree(8192);
    parms.set_coeff_modulus(CoeffModulus::BFVDefault(8192));
    parms.set_plain_modulus(1 << 20);
    
    SEALContext context(parms);
    KeyGenerator keygen(context);
    auto sk = keygen.secret_key();
    PublicKey pk;
    keygen.create_public_key(pk);
    
    Encryptor enc(context, pk);
    Evaluator eval(context);
    Decryptor dec(context, sk);
    BatchEncoder encoder(context);
    
    // 各人が暗号化した給与
    vector<Ciphertext> salaries;
    vector<int64_t> raw_salaries = {500000, 600000, 450000, 700000, 550000};
    for (auto s : raw_salaries) {
        vector<int64_t> vec(encoder.slot_count(), 0);
        vec[0] = s;   // 単一スロットだけ使う (デモ用)
        Plaintext pt; encoder.encode(vec, pt);
        Ciphertext ct; enc.encrypt(pt, ct);
        salaries.push_back(ct);
    }
    
    // 集計センターが暗号化のまま合計
    Ciphertext sum = salaries[0];
    for (size_t i = 1; i < salaries.size(); i++) {
        eval.add_inplace(sum, salaries[i]);
    }
    
    // 中央認証者が復号
    Plaintext result_pt; dec.decrypt(sum, result_pt);
    vector<int64_t> result; encoder.decode(result_pt, result);
    
    cout << "合計給与: " << result[0] << endl;
    cout << "平均給与: " << result[0] / raw_salaries.size() << endl;
}
```

個々の給与は中央認証者にも見えない（自分の鍵しか知らない）。合計のみを復号。

## 8. CKKS ML 推論の実装

### 8.1 簡単な線形回帰

OpenFHE Python で:

```python
from openfhe import *
import numpy as np

# モデル重み (平文)
weights = np.array([0.5, -0.3, 0.8, 0.1])
bias = 0.2

# 入力 x (暗号化)
parameters = CCParamsCKKSRNS()
parameters.SetMultiplicativeDepth(3)
parameters.SetScalingModSize(50)

cc = GenCryptoContext(parameters)
cc.Enable(PKE); cc.Enable(LEVELEDSHE)

keys = cc.KeyGen()
cc.EvalMultKeyGen(keys.secretKey)

x = [1.2, 0.5, -0.7, 2.1]
pt_x = cc.MakeCKKSPackedPlaintext(x)
ct_x = cc.Encrypt(keys.publicKey, pt_x)

# w * x を暗号化のまま計算
pt_w = cc.MakeCKKSPackedPlaintext(list(weights))
ct_prod = cc.EvalMult(ct_x, pt_w)

# スロット和 (rotation)
# ... (rotation で各スロット和を取る)

# bias 加算
ct_result = cc.EvalAdd(ct_prod, bias)

# 復号
result = cc.Decrypt(ct_result, keys.secretKey)
print(result)
```

### 8.2 ロジスティック回帰・NN

Concrete ML で。上記 5.4 参照。

### 8.3 実用例: MNIST

- **CKKS 実装**: LoLa (2019) で **0.3 秒 / 画像**
- **TFHE 実装**: Concrete ML の tutorial で数秒 / 画像

## 9. TFHE Boolean 回路

### 9.1 暗号化加算器

8 bit 加算を TFHE で実装:

```rust
use tfhe::prelude::*;
use tfhe::{ConfigBuilder, FheUint8, generate_keys, set_server_key};

fn full_adder(a: &FheBool, b: &FheBool, c_in: &FheBool) -> (FheBool, FheBool) {
    let sum = a ^ b ^ c_in;
    let c_out = (a & b) | (c_in & (a ^ b));
    (sum, c_out)
}

// TFHE-rs の高レベル API では FheUint8 が内部で展開する
fn main() {
    // 上記 4.2 と同様
    let a = FheUint8::encrypt(45u8, &client_key);
    let b = FheUint8::encrypt(30u8, &client_key);
    let c = &a + &b;
    // 内部で TFHE が 8 bit 加算器として回路展開
}
```

### 9.2 CSV 処理 (暗号化)

Concrete で暗号化 CSV 行を処理する例は公式 tutorial に。

## 10. ベンチマーク

### 10.1 典型的な測定

128-bit セキュリティ、$n = 8192$:

| 演算 | SEAL BFV | OpenFHE BFV | OpenFHE CKKS | TFHE-rs |
|---|---|---|---|---|
| Encrypt (1 ベクトル) | 1 ms | 1 ms | 2 ms | 2 ms (1 bit) |
| Add | 0.1 ms | 0.1 ms | 0.1 ms | 0.01 ms |
| Multiply | 5 ms | 4 ms | 3 ms | — |
| Relinearize | 10 ms | 8 ms | 8 ms | — |
| Rotate | 15 ms | 12 ms | 12 ms | — |
| Gate bootstrap | — | — | — | 13 ms |
| Full bootstrap | — | 20 s | 3 s | — |

### 10.2 ハードウェア依存

- **CPU (AVX-512)**: Intel HEXL で 2-5 倍加速
- **GPU**: CUDA 実装で 10 倍
- **FPGA/ASIC**: 100 倍〜（研究段階）

### 10.3 メモリ

- Ciphertext: 1 個で数 MB
- Public key: 数 MB
- Relin key, Galois keys: 合計 数十 MB 〜 数百 MB

大きなアプリではメモリが先にボトルネックになる。

## 11. デバッグ

### 11.1 よくあるエラー

**Noise budget exceeded**:
- 乗算しすぎた → `invariant_noise_budget()` で確認
- パラメータを大きくする、または再構造化

**Scale mismatch (CKKS)**:
- 乗算したら rescale を忘れた
- `rescale_to_next_inplace()` を追加

**Parms mismatch**:
- Level が違う暗号文を演算した
- `mod_switch_to_next_inplace()` で合わせる

**Segfault**:
- 対応する評価鍵が生成されていない
- `create_relin_keys`, `create_galois_keys` を忘れてないか確認

### 11.2 デバッグ手法

1. **Noise budget を頻繁にプリント**
2. **Intermediate decryption**: 開発中は中間結果を復号して確認
3. **Parameter scaling**: 小さい $n$ で動作確認 → 大きくしていく
4. **ログレベル**: OpenFHE の `SetLogLevel`

### 11.3 パフォーマンスチューニング

- **Batching**: 1 暗号文に $n$ スロット埋めて並列化
- **Minimize multiplications**: 加算と乗算のコスト比は 1:100 レベル
- **Lazy relinearization**: 連続乗算を 1 回にまとめる
- **Level 管理**: 最低限の $L$ で回す

## 12. Q&A

### Q1: どのライブラリから始めるべき？

**Python でお手軽** → Concrete (TFHE) or OpenFHE Python
**C++ で本格派** → SEAL
**最新研究追いたい** → OpenFHE

### Q2: ライブラリを混ぜて使える？

**暗号文形式が互換でないので基本不可**。**Scheme switching** という研究はあるが実装が限られる。

### Q3: 自作した方が早い？

**推奨しない**。FHE 実装は細部（noise, NTT, RNS）が複雑で、**実装バグ = 安全性崩壊** に直結。学習用に最小版を作るのは良いが、本番では既存ライブラリを。

### Q4: ハードウェア加速は使うべき？

**本番なら必須**。SEAL + HEXL (Intel CPU) は簡単に使える。GPU/FPGA は研究レベル。

### Q5: WebAssembly で動く？

**限定的に動く**。SEAL を Emscripten で WASM 化した例あり。ブラウザでの FHE 計算は未だ実用的ではない。

### Q6: ZKP と FHE のライブラリ連携は？

**研究段階**。**Verifiable FHE** 向けに、circom で FHE 復号回路を書いて ZKP と繋げる試みがある。

## 13. まとめ

### 本記事で学んだこと

- FHE の主要 6 ライブラリ: **SEAL, OpenFHE, HElib, TFHE-rs, Concrete, Lattigo**
- それぞれの **インストール・API** を把握
- **BFV 秘匿集計・CKKS ML 推論・TFHE Boolean 回路** の実装例
- **ベンチマーク** は秒〜ミリ秒オーダー（演算種類で大きく違う）
- **デバッグ**: noise budget、scale、level、evaluation key を意識
- **Python** が最も敷居が低い (Concrete, OpenFHE Python)

### 次の記事（Article 15）へ

最後の記事では **応用と展望** を扱う。現実のプロダクションで使われている FHE の実例、FHE ブロックチェーン、ZKP・MPC との組み合わせ、ハードウェア加速、標準化、そして DeIn のような実プロジェクトへの応用を総括する。シリーズ全体のまとめも含める。

### 3行サマリ

- **SEAL** が業界標準、**OpenFHE** が全スキーム網羅、**TFHE-rs/Concrete** は TFHE 専用
- Python なら **Concrete** が最も簡単、C++ なら **SEAL** が無難
- 実装は既存ライブラリに任せる。noise budget と level 管理がデバッグの肝

---

## 参考文献

- Microsoft SEAL. [https://github.com/microsoft/SEAL](https://github.com/microsoft/SEAL)
- OpenFHE. [https://github.com/openfheorg/openfhe-development](https://github.com/openfheorg/openfhe-development)
- HElib. [https://github.com/homenc/HElib](https://github.com/homenc/HElib)
- TFHE-rs. [https://github.com/zama-ai/tfhe-rs](https://github.com/zama-ai/tfhe-rs)
- Concrete. [https://github.com/zama-ai/concrete](https://github.com/zama-ai/concrete)
- Lattigo. [https://github.com/tuneinsight/lattigo](https://github.com/tuneinsight/lattigo)
- FHE.org Resources. [https://fhe.org/resources/](https://fhe.org/resources/)
