# 関数 (Functions)

このセクションでは、Sui Move の関数 (function) を紹介し、Hello World の例の一部として最初の Sui Move 関数を書きます。

## 関数の可視性 (Function Visibility)

Sui Move 関数には 3 つの可視性タイプがあります：

- **private**: 関数のデフォルトの可視性。同じモジュール内の関数のみがアクセスできます
- **public**: 同じモジュール内の関数と、他のモジュールで定義された関数からアクセスできます
- **public(package)**: 同じパッケージ内のモジュールの関数からアクセスできます

## 戻り値 (Return Value)

関数の戻り値の型は、関数のパラメータの後にコロンで区切って関数シグネチャー (function signature) で指定されます。

セミコロンのない関数の最後の行（実行の）が戻り値になります。

例：

```move
public fun addition (a: u8, b: u8): u8 {
    a + b
}
```

<!--
## Entry Functions

In Sui Move, entry functions are simply functions that can be called by transactions. They must satisfy the following three requirements:

- Denoted by the keyword `entry`
- have no return value
- (optional) have a mutable reference to an instance of the `TxContext` type in the last parameter

-->

## トランザクションコンテキスト (Transaction Context)

トランザクションから直接呼び出される関数は、通常最後のパラメータとして `TxContext` のインスタンスを持ちます。これは Sui Move VM によって設定される特別なパラメータで、関数を呼び出すユーザーが指定する必要はありません。

`TxContext` オブジェクトには、送信者のアドレス、tx のダイジェスト ID、tx のエポックなど、エントリー関数の呼び出しに使用されるトランザクションに関する[重要な情報](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/tx_context.move)が含まれています。

## `mint` 関数の作成

Hello World の例でミント関数 (minting function) を以下のように定義できます：

```move
public fun mint(ctx: &mut TxContext) {
    let object = HelloWorldObject {
        id: object::new(ctx),
        text: b"Hello World!".to_string()
    };
    transfer::public_transfer(object, ctx.sender());
}
```

この関数は単純に `HelloWorldObject` カスタムタイプの新しいインスタンスを作成し、Sui システムの [`public_transfer`](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/transfer.md#function-public_transfer) 関数を使用してトランザクション呼び出し元に送信します。
