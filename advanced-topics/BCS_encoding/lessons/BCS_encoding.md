# BCS エンコーディング

Binary Canonical Serialization（BCS）は、Diem ブロックチェーンのコンテキストで開発されたシリアライゼーション形式で、現在 Move（Sui、Starcoin、Aptos、0L）に基づくほとんどのブロックチェーンで広く使用されています。BCS は Move VM だけでなく、署名前のトランザクションのシリアライゼーションやイベントデータの解析など、トランザクションやイベントのコーディングにも使用されます。

BCS の動作原理を理解することは、Move をより深いレベルで理解し、Move エキスパートになりたい場合に重要です。詳しく見ていきましょう。

## BCS 仕様と特性

レッスンの残りを進めるにあたって、覚えておくとよい BCS エンコーディングの高レベルな特性がいくつかあります：

- BCS は、結果の出力バイトに型情報を含まないデータシリアライゼーション形式です。このため、エンコードされたバイトを受信する側は、データをデシリアライズする方法を知っている必要があります
- BCS には構造体がありません（型がないため）。構造体は単にフィールドがシリアライズされる順序を定義するだけです
- ラッパー型は無視されるため、`OuterType` と `UnnestedType` は同じ BCS 表現を持ちます：

  ```move
  public struct OuterType {
      owner: InnerType
  }
  public struct InnerType {
      address: address
  }
  public struct UnnestedType {
      address: address
  }
  ```

- ジェネリック型フィールドを含む型は、最初のジェネリック型フィールドまで解析できます。そのため、シリアライズ/デシリアライズされるカスタム型の場合、ジェネリック型フィールドを最後に配置するのが良い習慣です。
  ```move
  public struct BCSObject<T> has drop, copy {
      id: ID,
      owner: address,
      meta: Metadata,
      generic: T
  }
  ```
  この例では、`meta` フィールドまですべてをデシリアライズできます。
- 符号なし整数などのプリミティブ型は、リトルエンディアン形式でエンコードされます
- Vector は、[ULEB128](https://en.wikipedia.org/wiki/LEB128) 長さ（最大長さは `u32` まで）に続いてベクターの内容としてシリアライズされます。

完全な BCS 仕様は、[BCS リポジトリ](https://github.com/zefchain/bcs)で見つけることができます。

## `@mysten/bcs` JavaScript ライブラリの使用

### インストール

この部分で必要になるライブラリは、[@mysten/bcs ライブラリ](https://www.npmjs.com/package/@mysten/bcs)です。node プロジェクトのルートディレクトリで以下を入力してインストールできます：

```bash
npm i @mysten/bcs
```

### 基本例

まず、JavaScript ライブラリを使用して、いくつかの単純なデータ型をシリアライズおよびデシリアライズしましょう：

```javascript
import { bcs } from "@mysten/bcs";

// Define some test data types
const integer = 10;
const array = [1, 2, 3, 4];
const string = "test string";

// use .serialize() to serialize data
const ser_integer = bcs.u16().serialize(integer);
const ser_array = bcs.vector(bcs.u8()).serialize(array);
const ser_string = bcs.string().serialize(string);

// use .parse() to deserialize data
const de_integer = bcs.u16().parse(ser_integer.toBytes());
const de_array = bcs.vector(bcs.u8()).parse(ser_array.toBytes());
const de_string = bcs.string().parse(ser_string.toBytes());
```

シリアライザーは、上記の構文を使用して `@mysten/bcs` ライブラリから直接インポートできます。

Sui Move 型用に `bcs.u16()`、`bcs.string()` などの組み込みメソッドが使用できます。[ジェネリック型](../../../unit-three/lessons/2_intro_to_generics.md)については、ベクター用に `bcs.vector(bcs.u8())` などのメソッドが使用できます。

シリアライズおよびデシリアライズされたフィールドを詳しく見てみましょう：

```bash
# ints are little-endian hexadecimals
0a00
10
# The first element of a vector indicates the total length,
# then it's just whatever elements are in the vector
0401020304
1,2,3,4
# strings are just vectors of u8's, with the first element equal to the length of the string
0b7465737420737472696e67
test string
```

### 型登録

以下の構文を使用して、作業する予定のカスタム型を登録できます：

```javascript
import { bcs, fromHex, toHex } from "@mysten/bcs";

// Define Address as a 32-byte array, then add a transform to/from hex strings
const Address = bcs.fixedArray(32, bcs.u8()).transform({
  input: (id) => fromHex(id),
  output: (id) => toHex(Uint8Array.from(id)),
});

// Register the struct types
const bcsStruct = bcs.struct("BCSObject", {
  id: Address,
  owner: Address,
  meta: bcs.struct("Metadata", {
    name: bcs.string(),
  }),
});
```

## Sui スマートコントラクトでの `bcs` の使用

構造体を使用して、上記の例を続けましょう。

### 構造体定義

Sui Move コントラクトで対応する構造体定義から始めます。

```move
public struct Metadata has copy, drop {
    name: ascii::String,
}

public struct BCSObject has copy, drop {
    id: ID,
    owner: address,
    meta: Metadata,
}
```

### デシリアライゼーション

次に、Sui コントラクトでオブジェクトをデシリアライズする関数を書きましょう。

```move
public fun object_from_bytes(bcs_bytes: vector<u8>): BCSObject {
    let mut bcs = bcs::new(bcs_bytes);

    // Use `peel_*` functions to peel values from the serialized bytes.
    // Order has to be the same as we used in serialization!
    let (address, owner, meta) = (
        bcs.peel_address(),
        bcs.peel_address(),
        bcs.peel_vec_u8(),
    );
    // Pack a BCSObject struct with the results of serialization
    BCSObject {
        id: address.to_id(),
        owner,
        meta: Metadata { name: meta.to_ascii_string() },
    }
}
```

Sui フレームワークの [`bcs` モジュール](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/bcs.md)のさまざまな `peel_*` メソッドは、BCS シリアライズされたバイトから各個別フィールドを「剥がす (peel)」ために使用されます。フィールドを剥がす順序は、構造体定義のフィールドの順序と完全に同じでなければならないことに注意してください。

_クイズ：同じ `bcs` オブジェクトに対する最初の 2 つの `peel_address` 呼び出しの結果が同じでないのはなぜですか？_

また、`to_id()` を使用して `address` から `ID` に、`to_ascii_string()` を使用して `vector<u8>` から `ascii::String` に型を変換する方法にも注意してください。

_クイズ：`BCSObject` が `ID` 型ではなく `UID` 型を持っていた場合、何が起こるでしょうか？_

## 完全なシリアライゼーション/デシリアライゼーション例

完全な JavaScript と Sui Move のサンプルコードは、[`example_projects`](https://github.com/sui-foundation/sui-move-intro-course/tree/main/advanced-topics/BCS_encoding/example_projects) フォルダで見つけることができます。

まず、JavaScript プログラムを使用してテストオブジェクトをシリアライズします：

```javascript
import { bcs, fromHex, toHex } from "@mysten/bcs";

// Define Address as a 32-byte array, then add a transform to/from hex strings
const Address = bcs.fixedArray(32, bcs.u8()).transform({
  input: (id) => fromHex(id),
  output: (id) => toHex(Uint8Array.from(id)),
});

// We construct a test object to serialize
const bcsStruct = bcs.struct("BCSObject", {
  id: Address,
  owner: Address,
  meta: bcs.struct("Metadata", {
    name: bcs.string(),
  }),
});

const serialized = bcsStruct.serialize({
  id: "0x0000000000000000000000000000000000000000000000000000000000000005",
  owner: "0x000000000000000000000000000000000000000000000000000000000000000A",
  meta: {
    name: "aaa",
  },
});

console.log("Hex:", serialized.toHex());
```

`toHex()` メソッドを使用して、16 進数形式でシリアライゼーション結果を取得できます。

シリアライゼーション結果の 16 進文字列に `0x` プレフィックスを付けて、環境変数にエクスポートします：

```bash
export OBJECT_HEXSTRING=0x0000000000000000000000000000000000000000000000000000000000000005000000000000000000000000000000000000000000000000000000000000000a03616161
```

これで、関連する Move ユニットテストを実行して正確性を確認できます：

```bash
sui move test
```

コンソールに以下が表示されるはずです：

```bash
BUILDING bcs_move
Running the Move unit tests
[ PASS    ] 0x0::bcs_object::test_deserialization
Test result: OK. Total tests: 1; passed: 1; failed: 0
```

または、モジュールを公開し（PACKAGE_ID をエクスポート）、上記の BCS シリアライズされた 16 進文字列を使用して `emit_object` メソッドを呼び出すことができます：

```bash
sui client call --function emit_object --module bcs_object --package $PACKAGE_ID --args $OBJECT_HEXSTRING
```

その後、Sui Explorer のトランザクションの `Events` タブをチェックして、正しくデシリアライズされた `BCSObject` を発行したことを確認できます：

![Event](../images/event.png)
