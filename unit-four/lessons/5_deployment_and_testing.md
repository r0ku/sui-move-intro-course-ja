# デプロイメントとテスト

次に、SUI CLI を通じてマーケットプレイスコントラクトをデプロイ・テストできます。

テストを支援するために出品するアイテムをミントできるように、シンプルな `marketplace::widget` モジュールを作成します。

```move
module marketplace::widget;

public struct Widget has key, store {
    id: UID,
}

#[lint_allow(self_transfer)]
public fun mint(ctx: &mut TxContext) {
    let object = Widget {
        id: object::new(ctx),
    };
    transfer::public_transfer(object, ctx.sender());
}
```

これは基本的にユニット 1 の Hello World プロジェクトですが、さらにシンプルになっています。

## デプロイメント

以下を使用してパッケージを公開してください：

```bash
sui client publish
```

エクスプローラーで `marketplace` と `widget` の両方のモジュールが公開されているのを確認できるはずです：

![Publish](../images/publish.png)

パッケージオブジェクト ID を環境変数にエクスポートしてください：

```bash
export PACKAGE_ID=<前の出力からのパッケージオブジェクトID>
```

## マーケットプレイスの初期化

次に、`create` エントリー関数を呼び出してマーケットプレイスコントラクトを初期化する必要があります。このマーケットプレイスが受け入れるファンジブルトークンのタイプを指定するために型引数を渡したいと思います。ここでは `Sui` ネイティブトークンを使用するのが最も簡単です。以下の CLI コマンドを使用できます：

```bash
sui client call --function create --module marketplace --package $PACKAGE_ID --type-args 0x2::sui::SUI
```

`SUI` トークンの型引数を渡すための構文に注意してください。

`Marketplace` 共有オブジェクトの ID を環境変数にエクスポートしてください：

```bash
export MARKET_ID=<前の出力からのマーケットプレイス共有オブジェクトID>
```

## 出品

まず、出品する `widget` アイテムをミントします：

```bash
sui client call --function mint --module widget --package $PACKAGE_ID
```

ミントされた `widget` のオブジェクトアイテムを環境変数に保存してください：

```bash
export ITEM_ID=<コンソールからのウィジェットアイテムのオブジェクトID>
```

その後、このアイテムをマーケットプレイスに出品します：

```bash
sui client call --function list --module marketplace --package $PACKAGE_ID --args $MARKET_ID $ITEM_ID 1 --type-args $PACKAGE_ID::widget::Widget 0x2::sui::SUI
```

ここでは 2 つの型引数を送信する必要があります。最初は出品されるアイテムのタイプで、2 番目は支払いのファンジブルコインタイプです。上記の例では出品価格 `1` を使用しています。

このトランザクションを送信した後、[Sui explorer](https://suiexplorer.com/)で新しく作成されたリスティングを確認できます：

![Listing](../images/listing.png)

## 購入

支払いオブジェクトとして使用するために、金額 `1` の `SUI` コインオブジェクトを分割します。`sui client gas` CLI コマンドを使用して、アカウント下で利用可能な `SUI` コインのリストを確認し、分割するものを選択できます。

```bash
sui client split-coin --coin-id <分割するコインのオブジェクトID> --amounts 1
```

残高 `1` で新しく分割された `SUI` コインのオブジェクト ID をエクスポートしてください：

```bash
export PAYMENT_ID=<分割された残高1のSUIコインのオブジェクトID>
```

今度は、出品したばかりのアイテムを買い戻しましょう：

```bash
sui client call --function buy_and_take --module marketplace --package $PACKAGE_ID --args $MARKET_ID $ITEM_ID $PAYMENT_ID --type-args $PACKAGE_ID::widget::Widget 0x2::sui::SUI
```

このトランザクションを送信した後、コンソールでトランザクション効果の長いリストが表示されるはずです。`widget` が私たちのアドレスによって所有されており、`payments` `Table` に私たちのアドレスをキーとするエントリがあり、サイズが `1` であることを確認できます。

### 利益の取得

最後に、`take_profits_and_keep` メソッドを呼び出して収益を請求できます：

```bash
sui client call --function take_profits_and_keep --module marketplace --package $PACKAGE_ID --args $MARKET_ID --type-args 0x2::sui::SUI
```

これにより `payments` `Table` オブジェクトから残高が取得され、そのサイズが `0` に戻ります。エクスプローラーでこれを確認してください。
