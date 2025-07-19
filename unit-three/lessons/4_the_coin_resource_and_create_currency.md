# `Coin`リソースと`create_currency`メソッド

ジェネリックとウィットネスパターンがどのように動作するかがわかったので、`Coin` リソースと `create_currency` メソッドを見直しましょう。

## `Coin`リソース

ジェネリックがどのように動作するかを理解したので、`sui::coin` の `Coin` リソースを見直すことができます。以下のように[定義](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/coin.move#L40)されています：

```move
public struct Coin<phantom T> has key, store {
    id: UID,
    balance: Balance<T>
}
```

`Coin` リソースタイプは、ジェネリック型 `T` と 2 つのフィールド `id` と `balance` を持つ構造体です。`id` は `sui::object::UID` 型で、これは以前に見たことがあります。

`balance` は [`sui::balance::Balance`](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/balance.md#0x2_balance_Balance) 型で、以下のように[定義](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/balance.move#L31)されています：

```move
public struct Balance<phantom T> has store {
    value: u64
}
```

[`phantom`](./3_witness_design_pattern.md#the-phantom-keyword)に関する議論を思い出してください。型 `T` は `Coin` では `Balance` の別のファントム型への引数としてのみ使用され、`Balance` ではそのフィールドのいずれでも使用されていないため、`T` は `phantom` 型パラメータです。

`Coin<T>` は、アドレス間で転送したり、スマートコントラクト関数呼び出しで消費したりできるファンジブルトークン型 `T` の特定の量の転送可能なアセット表現として機能します。

## `create_currency`メソッド

`coin::create_currency` が実際に何を行っているかを[ソースコード](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/coin.move#L211)で見てみましょう：

```move
public fun create_currency<T: drop>(
    witness: T,
    decimals: u8,
    symbol: vector<u8>,
    name: vector<u8>,
    description: vector<u8>,
    icon_url: Option<Url>,
    ctx: &mut TxContext,
): (TreasuryCap<T>, CoinMetadata<T>) {
    // Make sure there's only one instance of the type T
    assert!(sui::types::is_one_time_witness(&witness), EBadWitness);

    (
        TreasuryCap {
            id: object::new(ctx),
            total_supply: balance::create_supply(witness),
        },
        CoinMetadata {
            id: object::new(ctx),
            decimals,
            name: string::utf8(name),
            symbol: ascii::string(symbol),
            description: string::utf8(description),
            icon_url,
        },
    )
}
```

assert は、Sui フレームワークの [`sui::types::is_one_time_witness`](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/types.move) メソッドを使用して、渡された `witness` リソースがワンタイムウィットネスであることをチェックします。

このメソッドは 2 つのオブジェクトを作成して返します。1 つは `TreasuryCap` リソース、もう 1 つは `CoinMetadata` リソースです。

### `TreasuryCap`

`TreasuryCap` はアセットであり、ワンタイムウィットネスパターンによってシングルトンオブジェクトであることが保証されています：

```move
/// Capability allowing the bearer to mint and burn
/// coins of type `T`. Transferable
public struct TreasuryCap<phantom T> has key, store {
    id: UID,
    total_supply: Supply<T>
}
```

これは `Balance::Supply` 型のシングルトンフィールド `total_supply` をラップします：

```move
/// A Supply of T. Used for minting and burning.
/// Wrapped into a `TreasuryCap` in the `Coin` module.
public struct Supply<phantom T> has store {
    value: u64
}
```

`Supply<T>` は、現在流通している型 `T` の指定されたカスタムファンジブルトークンの総量を追跡します。このフィールドがシングルトンでなければならない理由がわかります。単一のトークン型に対して複数の `Supply` インスタンスを持つことは意味がないからです。

### `CoinMetadata`

これは、作成されたファンジブルトークンのメタデータを格納するリソースです。以下のフィールドが含まれます：

- `decimals`: このカスタムファンジブルトークンの精度 (precision)
- `name`: このカスタムファンジブルトークンの名前
- `symbol`: このカスタムファンジブルトークンのトークンシンボル
- `description`: このカスタムファンジブルトークンの説明
- `icon_url`: このカスタムファンジブルトークンのアイコンファイルへの URL

`CoinMetadata` に含まれる情報は、Sui の基本的で軽量なファンジブルトークン標準と考えることができ、`sui::coin` モジュールを使用して作成されたファンジブルトークンをウォレットやエクスプローラーが表示するために使用できます。
