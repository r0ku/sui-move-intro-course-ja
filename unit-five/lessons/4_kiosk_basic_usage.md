# Kiosk の基本的な使用方法

## Kiosk の作成

まず、サンプルキオスクスマートコントラクトをデプロイし、後で使用するためにパッケージ ID をエクスポートしましょう。

```bash
export KIOSK_PACKAGE_ID=<サンプルキオスクスマートコントラクトのパッケージID>
```

```move
module kiosk::kiosk;
use sui::kiosk::{Self, Kiosk, KioskOwnerCap};

#[allow(lint(share_owned, self_transfer))]
/// Create new kiosk
public fun new_kiosk(ctx: &mut TxContext) {
    let (kiosk, kiosk_owner_cap) = kiosk::new(ctx);
    transfer::public_share_object(kiosk);
    transfer::public_transfer(kiosk_owner_cap, ctx.sender());
}
```

新しいキオスクを作成する方法は 2 つあります：

1. `kiosk::new()` を使用して新しいキオスクを作成しますが、`Kiosk` を共有オブジェクトにし、`sui::transfer` を使用して `KioskOwnerCap` を送信者に転送する必要があります。

```bash
sui client call --package $KIOSK_PACKAGE_ID --module kiosk --function new_kiosk
```

2. `entry kiosk::default()` を使用して、上記のすべてのステップを自動的に実行します。

後で使用するために、新しく作成された `Kiosk` とその `KioskOwnerCap` をエクスポートできます。

```bash
export KIOSK=<新しく作成されたKioskのオブジェクトID>
export KIOSK_OWNER_CAP=<新しく作成されたKioskOwnerCapのオブジェクトID>
```

_💡 注意：Kiosk はデフォルトで異質コレクション (heterogeneous collection) なので、アイテムに型パラメータが必要ありません。_

## Kiosk へのアイテム配置

```move
public struct TShirt has key, store {
    id: UID,
}

public fun new_tshirt(ctx: &mut TxContext): TShirt {
    TShirt {
        id: object::new(ctx),
    }
}

/// Place item inside kiosk
public fun place(kiosk: &mut Kiosk, cap: &KioskOwnerCap, item: TShirt) {
    kiosk.place(cap, item)
}
```

`kiosk::place()` API を使用してキオスク内にアイテムを配置できます。Kiosk Owner のみがこの API にアクセスできることを覚えておいてください。

## Kiosk からのアイテム引き出し

```move
/// Withdraw item from Kiosk
public fun withdraw(
    kiosk: &mut Kiosk,
    cap: &KioskOwnerCap,
    item_id: object::ID,
): TShirt {
    kiosk.take(cap, item_id)
}
```

`kiosk::take()` API を使用してキオスクからアイテムを引き出すことができます。Kiosk Owner のみがこの API にアクセスできることを覚えておいてください。

## 販売用アイテム出品

```move
/// List item for sale
public fun list(
    kiosk: &mut Kiosk,
    cap: &KioskOwnerCap,
    item_id: object::ID,
    price: u64,
) {
    kiosk.list<TShirt>(cap, item_id, price)
}
```

`kiosk::list()` API を使用してアイテムを販売用に出品できます。Kiosk Owner のみがこの API にアクセスできることを覚えておいてください。
