# 管理型コインの例

`sui::coin` モジュールの仕組みを詳しく見たので、多くの ERC-20 実装と同様に、ミント (mint) とバーン (burn) の機能を持つ信頼できる管理者がいるカスタムファンジブルトークンタイプを作成するシンプルで完全な例を見ることができます。

## スマートコントラクト

完全な[管理型コインサンプルコントラクト](../example_projects/fungible_tokens/sources/managed.move)は、サンプルプロジェクトフォルダの下で確認できます。

これまでに説明した内容を考えると、このコントラクトは理解しやすいはずです。これは[ワンタイムウィットネス](./3_witness_design_pattern.md#one-time-witness)パターンに正確に従っており、`witness` リソースは `MANAGED` という名前で、モジュールの `init` 関数によって自動的に作成されます。

`init` 関数は `coin::create_currency` を呼び出して `TreasuryCap` と `CoinMetadata` リソースを取得します。この関数に渡されるパラメータは `CoinMetadata` オブジェクトのフィールドなので、トークン名、シンボル、アイコン URL などが含まれます。

`CoinMetadata` は作成後すぐに `transfer::freeze_object` メソッドを介して凍結され、任意のアドレスが読み取ることができる[共有不変オブジェクト](../../unit-two/lessons/2_ownership.md#shared-immutable-objects)になります。

`TreasuryCap` [ケイパビリティ](../../unit-two/lessons/6_capability_design_pattern.md)オブジェクトは、それぞれ `Coin<MANAGED>` オブジェクトを作成または破棄する `mint` および `burn` メソッドへのアクセスを制御する方法として使用されます。

## 公開と CLI テスト

### モジュールの公開

[fungible_tokens](../example_projects/fungible_tokens/)プロジェクトフォルダの下で、以下を実行してください：

```bash
sui client publish
```

以下のようなコンソール出力が表示されるはずです：

![Publish Output](../images/publish.png)

作成された 2 つの不変オブジェクトは、それぞれパッケージ自体と `Managed Coin` の `CoinMetadata` オブジェクトです。そして、トランザクション送信者に渡される所有オブジェクトは `Managed Coin` の `TreasuryCap` オブジェクトです。

![Treasury Object](../images/treasury.png)

パッケージオブジェクトと `TreasuryCap` オブジェクトのオブジェクト ID を環境変数にエクスポートしてください：

```bash
export PACKAGE_ID=<前の出力からのパッケージオブジェクトID>
export TREASURYCAP_ID=<前の出力からのtreasury capオブジェクトID>
```

### トークンのミント

いくつかの `MNG` トークンをミントするには、以下の CLI コマンドを使用できます：

```bash
sui client call --function mint --module managed --package $PACKAGE_ID --args $TREASURYCAP_ID <ミントする量> <受信者アドレス>
```

![Minting](../images/minting.png)

新しくミントされた `COIN<MANAGED>` オブジェクトのオブジェクト ID を bash 変数にエクスポートしてください：

```bash
export COIN_ID=<前の出力からのコインオブジェクトID>
```

`TreasuryCap<MANAGED>` オブジェクトの下の `Supply` フィールドが、ミントされた量だけ増加しているかを確認してください。

### トークンのバーン

既存の `COIN<MANAGED>` オブジェクトをバーンするには、以下の CLI コマンドを使用します：

```bash
sui client call --function burn --module managed --package $PACKAGE_ID --args $TREASURYCAP_ID $COIN_ID
```

![Burning](../images/burning.png)

`TreasuryCap<MANAGED>` オブジェクトの下の `Supply` フィールドが `0` に戻っているかを確認してください。

_演習：ファンジブルトークンには他にどのような一般的に使用される関数が必要ですか？Move でのプログラミングについて十分に知識があるので、これらの関数のいくつかを実装してみてください。_
