# Clock とロック済みコインの例

2 番目のファンジブルトークンの例では、Sui でオンチェーン時刻を取得する方法と、それを利用してコインのベスティング機能 (vesting mechanism) を実装する方法を紹介します。

## Clock

Sui フレームワークには、Move スマートコントラクトでタイムスタンプを利用可能にするネイティブ[clock モジュール](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/clock.md)があります。

アクセスする必要がある主要なメソッドは以下の通りです：

```
public fun timestamp_ms(clock: &clock::Clock): u64
```

[`timestamp_ms`](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/clock.md#function-timestamp_ms) 関数は、過去の任意の時点からの実行中のミリ秒の合計として、現在のシステムタイムスタンプを返します。

[`clock`](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/clock.md#sui_clock_Clock) オブジェクトには特別な予約済み識別子 `0x6` があり、それを入力の 1 つとして使用する関数呼び出しに渡す必要があります。

## ロック済みコイン (Locked Coin)

`clock` を通じてオンチェーン時刻にアクセスする方法がわかったので、ベスティングファンジブルトークンの実装は比較的簡単です。

### `Locker` カスタムタイプ

`locked_coin` は `managed_coin` 実装の上に構築され、追加で 1 つのカスタムタイプ `Locker` があります：

```move
/// Transferrable object for storing the vesting coins
public struct Locker has key, store {
    id: UID,
    start_date: u64,
    final_date: u64,
    original_balance: u64,
    current_balance: Balance<LOCKED_COIN>

}
```

Locker は、発行されたトークンのベスティングスケジュール (vesting schedule) とベスティング状況に関する情報をエンコードする転送可能な[アセット](https://github.com/sui-foundation/sui-move-intro-course/blob/main/unit-one/lessons/3_custom_types_and_abilities.md#assets)です。

`start_date` と `final_date` は `clock` から取得されるタイムスタンプで、ベスティング期間の開始と終了をマークします。

`original_balance` は `Locker` に発行された初期残高、`balance` は既に引き出されたベスティング済み部分を考慮した現在の残り残高です。

### ミント

`locked_mint` メソッドでは、指定された量のトークンとエンコードされたベスティングスケジュールを持つ `Locker` を作成して転送します：

```move
/// Mints and transfers a locker object with the input amount of coins and
/// specified vesting schedule
public fun locked_mint(
    treasury_cap: &mut TreasuryCap<LOCKED_COIN>,
    recipient: address,
    amount: u64,
    lock_up_duration: u64,
    clock: &Clock,
    ctx: &mut TxContext,
) {
    let coin = treasury_cap.mint(amount, ctx);
    let start_date = clock.timestamp_ms();
    let final_date = start_date + lock_up_duration;

    transfer::public_transfer(
        Locker {
            id: object::new(ctx),
            start_date,
            final_date,
            original_balance: amount,
            current_balance: coin.into_balance(),
        },
        recipient,
    );
}
```

ここで `clock` がどのように現在のタイムスタンプを取得するために使用されているかに注目してください。

### 引き出し (Withdrawing)

`withdraw_vested` メソッドには、ベスティング済み量を計算するロジックの大部分が含まれています：

```move
/// Withdraw the available vested amount assuming linear vesting
public fun withdraw_vested(
    locker: &mut Locker,
    clock: &Clock,
    ctx: &mut TxContext,
) {
    let total_duration = locker.final_date - locker.start_date;
    let elapsed_duration = clock.timestamp_ms() - locker.start_date;
    let total_vested_amount = if (elapsed_duration > total_duration) {
        locker.original_balance
    } else {
        locker.original_balance * elapsed_duration / total_duration
    };
    let available_vested_amount =
        total_vested_amount - (locker.original_balance - locker.current_balance.value());
    transfer::public_transfer(
        coin::take(&mut locker.current_balance, available_vested_amount, ctx),
        ctx.sender(),
    )
}
```

この例ではシンプルな線形ベスティングスケジュールを想定していますが、幅広いベスティングロジックとスケジュールに対応するように変更できます。

### 完全なコントラクト

[`locked_coin`](../example_projects/locked_coin/sources/locked_coin.move) の実装の完全なスマートコントラクトは、[example_projects/locked_coin](../example_projects/locked_coin/) フォルダの下で確認できます。
