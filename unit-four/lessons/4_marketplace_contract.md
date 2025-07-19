# マーケットプレイスコントラクト

さまざまなタイプのコレクションと動的フィールドがどのように動作するかをしっかりと理解したので、以下の機能をサポートするオンチェーンマーケットプレイスのコントラクトを書き始めることができます：

- 任意のアイテムタイプと数量の出品
- カスタムまたはネイティブファンジブルトークンタイプでの支払い受付
- 複数の販売者が同時にアイテムを出品し、安全に支払いを受け取ることができる

## 型定義

まず、全体的な `Marketplace` 構造体を定義します：

```move
/// A shared `Marketplace`. Can be created by anyone using the
/// `create` function. One instance of `Marketplace` accepts
/// only one type of Coin - `COIN` for all its listings.
public struct Marketplace<phantom COIN> has key {
    id: UID,
    items: Bag,
    payments: Table<address, Coin<COIN>>
}
```

`Marketplace` は誰でもアクセス・変更できる共有オブジェクト (shared object) になります。支払いが受け入れられる[ファンジブルトークン](../../unit-three/lessons/4_the_coin_resource_and_create_currency.md)タイプを定義する `COIN` ジェネリック型パラメータを受け入れます。

`items` フィールドはアイテムリスティング (item listing) を保持し、これらは異なる型にできるため、ここでは異質な `Bag` コレクションを使用します。

`payments` フィールドは各販売者が受け取った支払いを保持します。これは販売者のアドレスをキーとし、受け入れられたコインタイプを値とするキーと値のペアで表すことができます。ここでのキーと値の型は同質で固定されているため、このフィールドには `Table` コレクションタイプを使用できます。

_クイズ：複数のファンジブルトークンタイプを受け入れるようにこの構造体をどのように変更しますか？_

次に、`Listing` タイプを定義します：

```move
/// A single listing that contains the listed item and its
/// price in [`Coin<COIN>`].
public struct Listing has key, store {
    id: UID,
    ask: u64,
    owner: address,
}
```

この構造体は、アイテム出品に関連する必要な情報を保持します。取引される実際のアイテムを動的オブジェクトフィールドとして `Listing` オブジェクトに添付し、アイテムフィールドやコレクションを定義する必要性を排除します。

`Listing` は `key` アビリティを持つため、コレクション内に配置する際にそのオブジェクト ID をキーとして使用できることに注意してください。

## 出品と出品取り消し

次に、アイテムの出品と出品取り消しのロジックを書きます。まず、アイテムの出品：

```move
/// List an item at the Marketplace.
public fun list<T: key + store, COIN>(
    marketplace: &mut Marketplace<COIN>,
    item: T,
    ask: u64,
    ctx: &mut TxContext,
) {
    let item_id = object::id(&item);
    let mut listing = Listing {
        ask,
        id: object::new(ctx),
        owner: ctx.sender(),
    };

    dof::add(&mut listing.id, true, item);
    marketplace.items.add(item_id, listing)
}
```

前述のとおり、動的オブジェクトフィールドインターフェース (dynamic object field interface) を使用して販売される任意の型のアイテムを添付し、次にアイテムのオブジェクト ID をキーとし、実際の `Listing` オブジェクトを値として（これが `Listing` も `store` アビリティを持つ理由です）、`Listing` オブジェクトをリスティングの `Bag` に追加します。

出品取り消しについては、以下のメソッドを定義します：

```move
/// Internal function to remove listing and get an item back. Only owner can do
/// that.
fun delist<T: key + store, COIN>(
    marketplace: &mut Marketplace<COIN>,
    item_id: ID,
    ctx: &TxContext,
): T {
    let Listing { mut id, owner, .. } = bag::remove(
        &mut marketplace.items,
        item_id,
    );

    assert!(ctx.sender() == owner, ENotOwner);

    let item = dof::remove(&mut id, true);
    id.delete();
    item
}

/// Call [`delist`] and transfer item to the sender.
public fun delist_and_take<T: key + store, COIN>(
    marketplace: &mut Marketplace<COIN>,
    item_id: ID,
    ctx: &mut TxContext,
) {
    let item = delist<T, COIN>(marketplace, item_id, ctx);
    transfer::public_transfer(item, ctx.sender());
}
```

出品取り消しされた `Listing` オブジェクトがアンパック・削除され、出品されたアイテムオブジェクトが [`dof::remove`](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/dynamic_object_field.move#L59) を通じて取得される方法に注意してください。Sui アセットは定義モジュールの外で破棄できないため、アイテムを出品取り消し者に転送する必要があります。

## 購入と支払い

アイテムの購入は出品取り消しと似ていますが、支払い処理の追加ロジックがあります。

```move
/// Internal function to purchase an item using a known Listing. Payment is done
/// in Coin<C>.
/// Amount paid must match the requested amount. If conditions are met,
/// owner of the item gets the payment and buyer receives their item.
fun buy<T: key + store, COIN>(
    marketplace: &mut Marketplace<COIN>,
    item_id: ID,
    paid: Coin<COIN>,
): T {
    let Listing {
        mut id,
        ask,
        owner,
    } = marketplace.items.remove(item_id);

    assert!(ask == paid.value(), EAmountIncorrect);

    // Check if there's already a Coin hanging and merge `paid` with it.
    // Otherwise attach `paid` to the `Marketplace` under owner's `address`.
    if (marketplace.payments.contains(owner)) {
        marketplace.payments.borrow_mut(owner).join(paid)
    } else {
        marketplace.payments.add(owner, paid)
    };

    let item = dof::remove(&mut id, true);
    id.delete();
    item
}

/// Call [`buy`] and transfer item to the sender.
public fun buy_and_take<T: key + store, COIN>(
    marketplace: &mut Marketplace<COIN>,
    item_id: ID,
    paid: Coin<COIN>,
    ctx: &mut TxContext,
) {
    transfer::public_transfer(
        buy<T, COIN>(marketplace, item_id, paid),
        ctx.sender(),
    )
}
```

最初の部分はリスティングからアイテムを出品取り消しするのと同じですが、送信された支払いが正しい金額かもチェックします。2 番目の部分では、支払いコインオブジェクトを `payments` `Table` に挿入し、販売者がすでに残高を持っているかどうかに応じて、シンプルな `table::add` を行うか、`table::borrow_mut` と `coin::join` を行って支払いを既存の残高にマージします。

エントリー関数 `buy_and_take` は単純に `buy` を呼び出し、購入されたアイテムを購入者に転送します。

### 利益の取得

最後に、販売者がマーケットプレイスから残高を取得するためのメソッドを定義します。

```move
/// Internal function to take profits from selling items on the `Marketplace`.
fun take_profits<COIN>(
    marketplace: &mut Marketplace<COIN>,
    ctx: &TxContext,
): Coin<COIN> {
    marketplace.payments.remove(ctx.sender())
}

#[lint_allow(self_transfer)]
/// Call [`take_profits`] and transfer Coin object to the sender.
public fun take_profits_and_keep<COIN>(
    marketplace: &mut Marketplace<COIN>,
    ctx: &mut TxContext,
) {
    transfer::public_transfer(
        take_profits(marketplace, ctx),
        ctx.sender(),
    )
}
```

_クイズ：このマーケットプレイス設計では、なぜ[ケイパビリティ](../../unit-two/lessons/6_capability_design_pattern.md)ベースのアクセス制御を使用する必要がないのでしょうか？ここでケイパビリティデザインパターンを実装できますか？それはマーケットプレイスにどのような特性を与えるでしょうか？_

## 完全なコントラクト

汎用マーケットプレイスの実装の完全なスマートコントラクトは、[`example_projects/marketplace`](../example_projects/marketplace/sources/marketplace.move) フォルダの下で確認できます。
