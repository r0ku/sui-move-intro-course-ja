# 【オプション】クローズドループトークン標準

クローズドループトークン (Closed Loop Token) は、コントラクトデプロイヤーがトークンの転送、使用、ミントなどの方法を制御するトークンポリシーを定義できる Sui トークン標準です。

## `Coin` と `Balance` との関係

![Trinity](../images/trinity.png)

## アクション (Action)

### パブリック

- `token::keep` - トランザクション送信者に Token を送信
- `token::join` - 2 つの Token を結合
- `token::split` - Token を 2 つに分割、分割する量を指定
- `token::zero` - 空（残高ゼロ）の Token を作成
- `token::destroy_zero` - 残高ゼロの Token を破棄

### プロテクテッド (Protected)

- `token::transfer` - 指定されたアドレスに Token を転送
- `token::to_coin` - Token を Coin に変換
- `token::from_coin` - Coin を Token に変換
- `token::spend` - 指定されたアドレスで Token を使用

### アクションリクエスト (Action Request)

プロテクテッドアクションは確認が必要な `ActionRequest` を生成します。

```move
public struct ActionRequest<phantom T> {
    /// Name of the Action to look up in the Policy. Name can be one of the
    /// default actions: `transfer`, `spend`, `to_coin`, `from_coin` or a
    /// custom action.
    name: String,
    /// Amount is present in all of the txs
    amount: u64,
    /// Sender is a permanent field always
    sender: address,
    /// Recipient is only available in `transfer` action.
    recipient: Option<address>,
    /// The balance to be "spent" in the `TokenPolicy`, only available
    /// in the `spend` action.
    spent_balance: Option<Balance<T>>,
    /// Collected approvals (stamps) from completed `Rules`. They're matched
    /// against `TokenPolicy.rules` to determine if the request can be
    /// confirmed.
    approvals: VecSet<TypeName>,
}
```

## アクションリクエストの確認

アクションリクエストを確認する方法は 3 つあります。

- `TreasuryCap` による確認
- `TokenPolicyCap` による確認
- 定義されたトークンポリシーを通じた確認

## トークンポリシーの設定

- `coin::create_currency` を通じて Coin を作成
- `token::new_policy` を通じて該当トークンのポリシーを作成
- `TokenPolicy` オブジェクトを共有
- いずれかまたはすべてのアクションタイプに対して該当ルールを作成
- `TokenPolicy` からルールを登録、変更、または削除

### 階層

`Coin/Token Type` -> `TokenPolicy` -> `Rules`

## パリティトークンの例

これは、トークンポリシーがどのように定義され使用されるかを示すシンプルなクローズドループトークンの例です。

この例では、トークンの奇数パリティ量でのみミントを許可します。

### トークンポリシーの定義と追加

特定のトークンポリシーは[parity_rule.move](../example_projects/closed_loop_token/sources/parity_rule.move)コントラクトで定義されています。

その後、このルールは[parity.move](../example_projects/closed_loop_token/sources/parity.move)コントラクトの `init` 関数で定義された `PARITY` トークンに追加されます。

### 完全なコントラクト

完全なパリティトークンサンプルプロジェクトはこちらで提供されています：[Parity Token](../example_projects/closed_loop_token/)
