# 同質コレクション (Homogeneous Collections)

Sui 上でマーケットプレイスを構築するという主要トピックに入る前に、まず Move でのコレクション (collection) について学びましょう。

## vector（ベクター）

Move の `Vector` は、C++などの他の言語のものと同様です。実行時に動的にメモリを割り当て、特定の型や[ジェネリック型](../../unit-three/lessons/2_intro_to_generics.md)である単一の型のグループを管理する方法です。

`vector` の定義とその基本操作については、含まれているサンプルコードを参照してください。

```move
module collection::vector;

public struct Widget {}

#[allow(unused_field)]
// Vector for a specified type
public struct WidgetVector {
    widgets: vector<Widget>,
}

// Vector for a generic type
public struct GenericVector<T> {
    values: vector<T>,
}

// Creates a GenericVector that hold a generic type T
public fun create<T>(): GenericVector<T> {
    GenericVector<T> {
        values: vector::empty<T>(),
    }
}

// Push a value of type T into a GenericVector
public fun put<T>(vec: &mut GenericVector<T>, value: T) {
    vec.values.push_back(value);
}

// Pops a value of type T from a GenericVector
public fun remove<T>(vec: &mut GenericVector<T>): T {
    vec.values.pop_back()
}

// Returns the size of a given GenericVector
public fun size<T>(vec: &mut GenericVector<T>): u64 {
    vec.values.length()
}
```

ジェネリック型で定義されたベクターは*任意の型*のオブジェクトを受け入れることができますが、コレクション内のすべてのオブジェクトは依然として*同じ型*でなければならず、つまりコレクションは*同質 (homogeneous)* であることが重要です。

## Table

`Table` はキーと値のペアを動的に格納するマップのようなコレクションです。しかし、従来のマップコレクションとは異なり、そのキーと値は `Table` 値の中に格納されるのではなく、代わりに Sui のオブジェクトシステムを使用して格納されます。`Table` 構造体は、それらのキーと値を取得するためのオブジェクトシステムへのハンドルとしてのみ機能します。

`Table` の `key` 型は `copy + drop + store` のアビリティ制約を持つ必要があり、`value` 型は `store` のアビリティ制約を持つ必要があります。

`Table` も*同質*コレクションの一種で、キーと値のフィールドは指定された型またはジェネリック型にできますが、`Table` コレクション内のすべての値とすべてのキーは*同じ*型でなければなりません。

_クイズ：まったく同じキーと値のペアを含む 2 つのテーブルオブジェクトは、`===` 演算子でチェックした場合、互いに等しいでしょうか？試してみてください。_

`Table` コレクションの操作については、以下の例を参照してください：

```move
module collection::table;

use sui::table::{Self, Table};

#[allow(unused_field)]
// Defining a table with specified types for the key and value
public struct IntegerTable {
    table_values: Table<u8, u8>,
}

// Defining a table with generic types for the key and value
public struct GenericTable<phantom K: copy + drop + store, phantom V: store> {
    table_values: Table<K, V>,
}

// Create a new, empty GenericTable with key type K, and value type V
public fun create<K: copy + drop + store, V: store>(
    ctx: &mut TxContext,
): GenericTable<K, V> {
    GenericTable<K, V> {
        table_values: table::new<K, V>(ctx),
    }
}

// Adds a key-value pair to GenericTable
public fun add<K: copy + drop + store, V: store>(
    table: &mut GenericTable<K, V>,
    k: K,
    v: V,
) {
    table.table_values.add(k, v);
}

/// Removes the key-value pair in the GenericTable `table: &mut Table<K, V>` and
/// returns the value.
public fun remove<K: copy + drop + store, V: store>(
    table: &mut GenericTable<K, V>,
    k: K,
): V {
    table.table_values.remove(k)
}

// Borrows an immutable reference to the value associated with the key in
// GenericTable
public fun borrow<K: copy + drop + store, V: store>(
    table: &GenericTable<K, V>,
    k: K,
): &V {
    table.table_values.borrow(k)
}

/// Borrows a mutable reference to the value associated with the key in
/// GenericTable
public fun borrow_mut<K: copy + drop + store, V: store>(
    table: &mut GenericTable<K, V>,
    k: K,
): &mut V {
    table.table_values.borrow_mut(k)
}

/// Check if a value associated with the key exists in the GenericTable
public fun contains<K: copy + drop + store, V: store>(
    table: &GenericTable<K, V>,
    k: K,
): bool {
    table.table_values.contains(k)
}

/// Returns the size of the GenericTable, the number of key-value pairs
public fun length<K: copy + drop + store, V: store>(
    table: &GenericTable<K, V>,
): u64 {
    table.table_values.length()
}
```
