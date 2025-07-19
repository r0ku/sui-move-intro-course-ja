# 異質コレクション (Heterogeneous Collections)

`Vector` や `Table` のような同質コレクション (homogeneous collection) は、同じ型のオブジェクトのコレクションを保持する必要があるマーケットプレイス（または他のタイプのアプリケーション）では機能しますが、異なる型のオブジェクトを保持する必要がある場合や、保持する必要があるオブジェクトの型がコンパイル時にわからない場合はどうでしょうか？

このタイプのマーケットプレイスでは、販売されるアイテムを保持するために*異質 (heterogeneous)* コレクションを使用する必要があります。動的フィールドの理解という重労働をすでに終えているので、Sui での異質コレクションは理解しやすいはずです。ここでは `Bag` コレクションタイプをより詳しく見ていきます。

## `Bag` タイプ

`Bag` は異質マップライクコレクションです。このコレクションは、そのキーと値が `Bag` 値内に格納されるのではなく、代わりに Sui のオブジェクトシステムを使用して格納される点で `Table` に似ています。`Bag` 構造体は、それらのキーと値を取得するためのオブジェクトシステムへのハンドルとしてのみ機能します。

### 一般的な `Bag` 操作

一般的な `Bag` 操作のサンプルコードを以下に示します：

```move
module collection::bag;

use sui::bag::{Self, Bag};

// Defining a table with generic types for the key and value
public struct GenericBag {
    items: Bag,
}

// Create a new, empty GenericBag
public fun create(ctx: &mut TxContext): GenericBag {
    GenericBag {
        items: bag::new(ctx),
    }
}

/// Adds a key-value pair to GenericBag
public fun add<K: copy + drop + store, V: store>(
    bag: &mut GenericBag,
    k: K,
    v: V,
) {
    bag.items.add(k, v);
}

/// Removes the key-value pair from the GenericBag with the provided key and
/// returns the value.
public fun remove<K: copy + drop + store, V: store>(
    bag: &mut GenericBag,
    k: K,
): V {
    bag.items.remove(k)
}

// Borrows an immutable reference to the value associated with the key in
// GenericBag
public fun borrow<K: copy + drop + store, V: store>(
    bag: &GenericBag,
    k: K,
): &V {
    bag.items.borrow(k)
}

/// Borrows a mutable reference to the value associated with the key in
/// GenericBag
public fun borrow_mut<K: copy + drop + store, V: store>(
    bag: &mut GenericBag,
    k: K,
): &mut V {
    bag.items.borrow_mut(k)
}

/// Check if a value associated with the key exists in the GenericBag
public fun contains<K: copy + drop + store>(bag: &GenericBag, k: K): bool {
    bag.items.contains(k)
}

/// Returns the size of the GenericBag, the number of key-value pairs
public fun length(bag: &GenericBag): u64 {
    bag.items.length()
}
```

Bag コレクションとの相互作用のための関数シグネチャー (function signature) は、Table コレクションとの相互作用のための関数シグネチャーと非常に似ています。主な違いは、新しい Bag を作成する際に型を宣言する必要がなく、Bag に追加するキーと値のペアが異なる型にできることです。
