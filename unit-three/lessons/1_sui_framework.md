# Sui フレームワーク

スマートコントラクトの一般的な使用例は、カスタムファンジブルトークン（Ethereum の ERC-20 トークンなど）の発行です。Sui フレームワークを使用して Sui 上でそれを行う方法と、従来のファンジブルトークンのいくつかのバリエーションを見てみましょう。

## Sui フレームワーク

[Sui フレームワーク](https://github.com/MystenLabs/sui/tree/main/crates/sui-framework/docs)は、Sui の Move VM の固有実装です。Move 標準ライブラリの実装を含む Sui のネイティブ API と、[暗号プリミティブ (crypto primitive)](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/groth16.md)やフレームワークレベルでの[データ構造](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/url.md)の Sui 実装などの Sui 固有の操作が含まれています。

Sui でのカスタムファンジブルトークンの実装は、Sui フレームワークのライブラリの一部を大幅に活用します。

## `sui::coin`

Sui 上でカスタムファンジブルトークンを実装するために使用する主要なライブラリは、[`sui::coin`](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/coin.md)モジュールです。

ファンジブルトークンの例で直接使用するリソース (resource) またはメソッドは以下の通りです：

- リソース: [Coin](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/coin.md#resource-coin)
- リソース: [TreasuryCap](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/coin.md#resource-treasurycap)
- リソース: [CoinMetadata](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/coin.md#resource-coinmetadata)
- メソッド: [coin::create_currency](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/coin.md#0x2_coin_create_currency)

次の数セクションでいくつかの新しい概念を紹介した後、これらのそれぞれをより詳しく見直します。
