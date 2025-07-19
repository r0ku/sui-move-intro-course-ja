# TransferPolicy と Kiosk からの購入

このセクションでは、`TransferPolicy` を作成し、購入されたアイテムが購入者によって所有される前に購入者が準拠しなければならないルールを実施するために使用する方法を学びます。

## `TransferPolicy`

### `TransferPolicy` の作成

型 `T` の `TransferPolicy` は、その型 `T` が Kiosk システムで取引可能になるために作成されなければなりません。`TransferPolicy` は、購入されたアイテムが購入者に転送される前に、購入が定義されたポリシーに対して有効であることをすべての人にチェックすることを強制する中央権威として機能する共有オブジェクトです。

```move
use sui::transfer_policy::{Self, TransferRequest, TransferPolicy, TransferPolicyCap};
use sui::package::{Self, Publisher};

public struct KIOSK has drop {}

fun init(witness: KIOSK, ctx: &mut TxContext) {
    let publisher = package::claim(otw, ctx);
    transfer::public_transfer(publisher, ctx.sender());
}

#[allow(lint(share_owned, self_transfer))]
/// Create new policy for type `T`
public fun new_policy(publisher: &Publisher, ctx: &mut TxContext) {
    let (policy, policy_cap) = transfer_policy::new<TShirt>(publisher, ctx);
    transfer::public_share_object(policy);
    transfer::public_transfer(policy_cap, ctx.sender());
}
```

`TransferPolicy<T>` の作成には、`T` を含むモジュールの公開者証明 `Publisher` が必要です。これにより、型 `T` の作成者のみが `TransferPolicy<T>` を作成できることが保証されます。ポリシーを作成する方法は 2 つあります：

- `transfer_policy::new()` を使用して新しいポリシーを作成し、`TransferPolicy` を共有オブジェクトにし、`sui::transfer` を使用して `TransferPolicyCap` を送信者に転送する。

```bash
sui client call --package $KIOSK_PACKAGE_ID --module kiosk --function new_policy --args $KIOSK_PUBLISHER
```

- `entry transfer_policy::default()` を使用して、上記のすべてのステップを自動的に実行する。

パッケージを公開する際に `Publisher` オブジェクトをすでに受け取っているはずです。後で使用するためにエクスポートしましょう。

```bash
export KIOSK_PUBLISHER=<PublisherオブジェクトID>
```

ターミナルで新しく作成された `TransferPolicy` オブジェクトと `TransferPolicyCap` オブジェクトが表示されるはずです。後で使用するためにエクスポートしましょう。

```bash
export KIOSK_TRANSFER_POLICY=<TransferPolicyオブジェクトID>
export KIOSK_TRANSFER_POLICY_CAP=<TransferPolicyCapオブジェクトID>
```

### 固定料金ルールの実装

`TransferPolicy` はルールなしでは何も実施しません。取引を成功させるためにユーザーに固定ロイヤリティ料金の支払いを強制する簡単なルールを別のモジュールで実装する方法を学びましょう。

_💡 注意：ルールを実装するための標準的なアプローチがあります。[こちらのルールテンプレート](../example_projects/kiosk/sources/dummy_policy.move)をチェックしてください。_

#### ルールウィットネスとルール設定

```move
module kiosk::fixed_royalty_rule;

/// The `amount_bp` passed is more than 100%.
const EIncorrectArgument: u64 = 0;
/// The `Coin` used for payment is not enough to cover the fee.
const EInsufficientAmount: u64 = 1;

/// Max value for the `amount_bp`.
const MAX_BPS: u16 = 10_000;

/// The Rule Witness to authorize the policy
public struct Rule has drop {}

/// Configuration for the Rule
public struct Config has store, drop {
    /// Percentage of the transfer amount to be paid as royalty fee
    amount_bp: u16,
    /// This is used as royalty fee if the calculated fee is smaller than `min_amount`
    min_amount: u64,
}
```

`Rule` は `TransferPolicy` に追加するウィットネス型を表し、1 つのポリシーに追加される複数のルール間を識別し区別するのに役立ちます。`Config` は `Rule` の設定で、固定ロイヤリティ料金を実装するため、設定には元の支払いから差し引きたいパーセンテージを含める必要があります。

#### TransferPolicy へのルール追加

```move
/// Function that adds a Rule to the `TransferPolicy`.
/// Requires `TransferPolicyCap` to make sure the rules are
/// added only by the publisher of T.
public fun add<T>(
    policy: &mut TransferPolicy<T>,
    cap: &TransferPolicyCap<T>,
    amount_bp: u16,
    min_amount: u64

) {
    assert!(amount_bp <= MAX_BPS, EIncorrectArgument);
    transfer_policy::add_rule(
        Rule {},
        policy,
        cap,
        Config { amount_bp, min_amount },
    )
}
```

`transfer_policy::add_rule()` を使用して、ルールとその設定をポリシーに追加します。

クライアントからこの関数を実行して `Rule` を `TransferPolicy` に追加しましょう。そうしなければ無効化されています。この例では、ロイヤリティ料金のパーセンテージを `0.1%` ～ `10 ベーシスポイント` に設定し、最小ロイヤリティ料金を `100 MIST` に設定します。

```bash
sui client call --package $KIOSK_PACKAGE_ID --module fixed_royalty_rule --function add --args $KIOSK_TRANSFER_POLICY $KIOSK_TRANSFER_POLICY_CAP 10 100 --type-args $KIOSK_PACKAGE_ID::kiosk::TShirt
```

#### ルールの満足

```move
/// Buyer action: Pay the royalty fee for the transfer.
public fun pay<T: key + store>(
    policy: &mut TransferPolicy<T>,
    request: &mut TransferRequest<T>,
    payment: Coin<SUI>,
) {
    let paid = transfer_policy::paid(request);
    let amount = fee_amount(policy, paid);

    assert!(payment.value() == amount, EInsufficientAmount);

    transfer_policy::add_to_balance(Rule {}, policy, payment);
    transfer_policy::add_receipt(Rule {}, request)
}

/// Helper function to calculate the amount to be paid for the transfer.
/// Can be used dry-runned to estimate the fee amount based on the Kiosk listing price.
public fun fee_amount<T: key + store>(
    policy: &TransferPolicy<T>,
    paid: u64,
): u64 {
    let config: &Config = transfer_policy::get_rule(Rule {}, policy);
    let mut amount = (
        ((paid as u128) * (config.amount_bp as u128) / 10_000) as u64,
    );

    // If the amount is less than the minimum, use the minimum
    if (amount < config.min_amount) {
        amount = config.min_amount
    };

    amount
}
```

ポリシーと支払い額を与えられたロイヤリティ料金を計算するヘルパー `fee_amount()` が必要です。`transfer_policy::get_rule()` を使用して設定を照会し、料金計算に使用します。

`pay()` は、`transfer_policy::confirm_request()` の前に `TransferRequest`（次のセクションで説明）を満たすためにユーザー自身が呼び出さなければならない関数です。`transfer_policy::paid()` は `TransferRequest` によって表される取引の元の支払いを提供します。ロイヤリティ料金計算後、`transfer_policy::add_to_balance()` を通じてポリシーに料金を追加します。ポリシーによって回収されたすべての料金はここに蓄積され、`TransferPolicyCap` 所有者は後で引き出すことができます。最後に、`transfer_policy::add_receipt()` を使用して、このルールが通過し `transfer_policy::confirm_request()` で確認される準備ができていることを `TransferRequest` にフラグを立てます。

## Kiosk からのアイテム購入

```move
use sui::transfer_policy::{Self, TransferRequest, TransferPolicy};

/// Buy listed item
public fun buy(
    kiosk: &mut Kiosk,
    item_id: object::ID,
    payment: Coin<SUI>,
): (TShirt, TransferRequest<TShirt>) {
    kiosk.purchase(item_id, payment)
}

/// Confirm the TransferRequest
public fun confirm_request(
    policy: &TransferPolicy<TShirt>,
    req: TransferRequest<TShirt>,
) {
    policy.confirm_request(req);
}
```

購入者が `kiosk::purchase()` API を使用してアセットを購入すると、アイテムが `TransferRequest` と一緒に返されます。`TransferRequest` は `transfer_policy::confirm_request()` を通じてそれを消費することを強制するホットポテトです。`transfer_policy::confirm_request()` の仕事は、`TransferPolicy` で設定・有効化されたすべてのルールがユーザーによって準拠されているかを検証することです。有効化されたルールのいずれかが満たされていない場合、`transfer_policy::confirm_request()` はエラーを投げ、トランザクションの失敗につながります。結果として、`transfer_policy::confirm_request()` の前にアイテムをアカウントに転送しようとしても、アイテムはあなたの所有下にありません。

_💡 注意：ユーザーは `confirm_request()` 呼び出しの前に TransferRequest が有効であることを保証するために、必要なすべての呼び出しで PTB を構成する必要があります。_

フローは以下のように図示できます：

_購入者 -> `kiosk::purchase()` -> `アイテム` + `TransferRequest` -> TransferRequest を満たすための後続の呼び出し -> `transfer_policy::confirm_request()` -> 所有権下でのアイテム転送_

## Kiosk 完全フローの例

前のセクションから、アイテムはキオスク内に配置され、販売可能になるために出品されなければならないことを思い出してください。アイテムがすでに価格 `10_000 MIST` で出品されていると仮定して、出品されたアイテムをターミナル変数としてエクスポートしましょう。

```bash
export KIOSK_TSHIRT=<出品されたTShirtのオブジェクトID>
```

取引を実行するための PTB を構築しましょう。フローは簡単で、キオスクから出品されたアイテムを購入し、アイテムと `TransferRequest` が返され、次に `fixed_royalty_fee::pay` を呼び出して `TransferRequest` を満たし、最終的にアイテムを購入者に転送する前に `confirm_request()` で `TransferRequest` を確認します。

```bash
sui client ptb \
--assign price 10000 \
--split-coins gas "[price]" \
--assign coin \
--move-call $KIOSK_PACKAGE_ID::kiosk::buy @$KIOSK @$KIOSK_TSHIRT coin.0 \
--assign buy_res \
--move-call $KIOSK_PACKAGE_ID::fixed_royalty_rule::fee_amount "<$KIOSK_PACKAGE_ID::kiosk::TShirt>" @$KIOSK_TRANSFER_POLICY price \
--assign fee_amount \
--split-coins gas "[fee_amount]"\
--assign coin \
--move-call $KIOSK_PACKAGE_ID::fixed_royalty_rule::pay "<$KIOSK_PACKAGE_ID::kiosk::TShirt>" @$KIOSK_TRANSFER_POLICY buy_res.1 coin.0 \
--move-call $KIOSK_PACKAGE_ID::kiosk::confirm_request  @$KIOSK_TRANSFER_POLICY buy_res.1 \
--move-call 0x2::transfer::public_transfer "<$KIOSK_PACKAGE_ID::kiosk::TShirt>" buy_res.0 <購入者アドレス> \

```
