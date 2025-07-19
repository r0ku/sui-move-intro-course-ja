# ユニットテスト (Unit Testing)

Sui は[Move テストフレームワーク](https://move-book.com/move-basics/testing)をサポートしています。ここでは、ユニットテストの書き方と実行方法を示すために、`Managed Coin` のユニットテストをいくつか作成します。

## テスト環境

Sui Move のテストコードは他の Sui Move コードと同じですが、実際のプロダクションコードと区別するための特別なアノテーション (annotation) と関数があります。
テスト関数またはモジュールは `#[test]` または `#[test_only]` アノテーションで始まります。

```move
#[test_only]
module fungible_tokens::managed_tests;

#[test]
fun mint_burn() {
}
```

`Managed Coin` のユニットテストを `managed_tests` という別のテストモジュールに配置します。

このモジュール内の各関数は、1 つまたは複数のトランザクションからなる 1 つのユニットテストと見なすことができます。`mint_burn` という 1 つのユニットテストを書きます。

## テストシナリオ (Test Scenario)

テスト環境内では、主に[`test_scenario` パッケージ](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/test/test_scenario.move)を活用してランタイム環境をシミュレートします。ここで理解し、相互作用する必要がある主要なオブジェクトは `Scenario` オブジェクトです。`Scenario` は複数トランザクションのシーケンスをシミュレートし、送信者アドレスで以下のように初期化できます：

```move
// Initialize a mock sender address
let addr1 = @0xA;
// Begins a multi-transaction scenario with addr1 as the sender
let mut scenario = test_scenario::begin(addr1);
...
// Cleans up the scenario object
scenario.end();
```

_💡`Scenario` オブジェクトはドロップできないため、`test_scenario::end` を使用してそのスコープの最後で明示的にクリーンアップする必要があることに注意してください。_

### モジュール状態の初期化

`Managed Coin` モジュールをテストするには、まずモジュール状態を初期化する必要があります。モジュールに `init` 関数があるため、まず `managed` モジュール内に `test_only` init 関数を作成する必要があります：

```move
#[test_only]
/// Wrapper of module initializer for testing
public fun test_init(ctx: &mut TxContext) {
    init(MANAGED {}, ctx)
}
```

これは基本的にテストでのみ使用できるモック `init` 関数です。その後、この関数を呼び出すだけで、シナリオでランタイム状態を初期化できます：

```move
// Run the managed coin module init function
{
    managed::test_init(scenario.ctx())
};
```

### ミント

[`next_tx` メソッド](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/test/test_scenario.move#L249)を使用して、`Coin<MANAGED>` オブジェクトをミントしたいシナリオの次のトランザクションに進みます。

これを行うには、まず `TreasuryCap<MANAGED>` オブジェクトを抽出する必要があります。シナリオからこれを取得するために `take_from_sender` という特別なテスト関数を使用します。取得しようとするオブジェクトの型パラメータを `take_from_sender` に渡す必要があることに注意してください。

その後、必要なパラメータをすべて使用して `managed::mint` を呼び出すだけです。

このトランザクションの最後に、`test_scenario::return_to_address` を使用して `TreasuryCap<MANAGED>` オブジェクトを送信者アドレスに返す必要があります。

```move
scenario.next_tx(addr1);
{
    let mut treasurycap = scenario.take_from_sender<TreasuryCap<MANAGED>>();
    managed::mint(&mut treasurycap, 100, addr1, scenario.ctx());
    test_scenario::return_to_address<TreasuryCap<MANAGED>>(
        addr1,
        treasurycap,
    );
};
```

### バーン

トークンのバーンをテストするには、手順はミントのテストと非常に似ています。唯一の違いは、ミントされた人から `Coin<MANAGED>` オブジェクトも取得する必要があることです。

## ユニットテストの実行

完全な[`managed_tests`](../example_projects/fungible_tokens/tests/managed_tests.move)モジュールのソースコードは、`example_projects/fungible_tokens/tests/` フォルダの下で確認できます。

ユニットテストを実行するには、CLI でプロジェクトディレクトリに移動し、以下のコマンドを入力してください：

```bash
sui move test
```

どのユニットテストが合格または失敗したかを示すコンソール出力が表示されるはずです。

![Unit Test](../images/unittest.png)
