# 動的フィールド (Dynamic Fields)

`Table` などのコレクションが Sui Move で実際にどのように実装されているかを詳しく理解するには、Sui Move の動的フィールド (dynamic field) の概念を紹介する必要があります。動的フィールドは、実行時に追加または削除でき、任意のユーザー指定名を持つことができる異質フィールド (heterogeneous field) です。

動的フィールドには 2 つのサブタイプがあります：

- **動的フィールド (Dynamic Fields)** は `store` アビリティを持つ任意の値を格納できますが、この種のフィールドに格納されたオブジェクトはラップされたと見なされ、ストレージにアクセスする外部ツール（エクスプローラー、ウォレットなど）によってその ID を介して直接アクセスすることはできません。
- **動的オブジェクトフィールド (Dynamic Object Fields)** の値は Sui オブジェクト（`key` と `store` のアビリティを持ち、最初のフィールドとして `id: UID` を持つ）でなければ*なりませんが*、アタッチ後もそのオブジェクト ID を介して直接アクセス可能です。

## 動的フィールド操作

### 動的フィールドの追加

動的フィールドの操作方法を説明するために、以下の構造体を定義します：

```move
// Parent struct
public struct Parent has key {
    id: UID,
}

// Dynamic field child struct type containing a counter
public struct DFChild has store {
    count: u64
}

// Dynamic object field child struct type containing a counter
public struct DOFChild has key, store {
    id: UID,
    count: u64,
}
```

オブジェクトに**動的フィールド**または**動的オブジェクトフィールド**を追加するための API は以下の通りです：

```move
module collection::dynamic_fields ;

use sui::dynamic_field as df;
use sui::dynamic_object_field as dof;

// Adds a DFChild to the parent object under the provided name
public fun add_dfchild(parent: &mut Parent, child: DFChild, name: vector<u8>) {
    df::add(&mut parent.id, name, child);
}

// Adds a DOFChild to the parent object under the provided name
public fun add_dofchild(
    parent: &mut Parent,
    child: DOFChild,
    name: vector<u8>,
) {
    dof::add(&mut parent.id, name, child);
}
```

### 動的フィールドのアクセスと変更

動的フィールドと動的オブジェクトフィールドは以下のように読み取りまたはアクセスできます：

```move
// Borrows a reference to a DOFChild
public fun borrow_dofchild(child: &DOFChild): &DOFChild {
    child
}

// Borrows a reference to a DFChild via its parent object
public fun borrow_dfchild_via_parent(
    parent: &Parent,
    child_name: vector<u8>,
): &DFChild {
    df::borrow<vector<u8>, DFChild>(&parent.id, child_name)
}

// Borrows a reference to a DOFChild via its parent object
public fun borrow_dofchild_via_parent(
    parent: &Parent,
    child_name: vector<u8>,
): &DOFChild {
    dof::borrow<vector<u8>, DOFChild>(&parent.id, child_name)
}
```

動的フィールドと動的オブジェクトフィールドは以下のように変更することもできます：

```move
// Mutate a DOFChild directly
public fun mutate_dofchild(child: &mut DOFChild) {
    child.count = child.count + 1;
}

// Mutate a DFChild directly
public fun mutate_dfchild(child: &mut DFChild) {
    child.count = child.count + 1;
}

// Mutate a DFChild's counter via its parent object
public fun mutate_dfchild_via_parent(
    parent: &mut Parent,
    child_name: vector<u8>,
) {
    let child = df::borrow_mut<vector<u8>, DFChild>(
        &mut parent.id,
        child_name,
    );
    child.count = child.count + 1;
}
```

_クイズ：なぜ `mutate_dofchild` はエントリー関数になれるのに `mutate_dfchild` はなれないのでしょうか？_

### 動的フィールドの削除

親オブジェクトから動的フィールドを以下のように削除できます：

```move
// Removes a DFChild given its name and parent object's mutable reference, and
// returns it by value
public fun remove_dfchild(
    parent: &mut Parent,
    child_name: vector<u8>,
): DFChild {
    df::remove<vector<u8>, DFChild>(&mut parent.id, child_name)
}

// Removes a DOFChild given its name and parent object's mutable reference, and
// returns it by value
public fun remove_dofchild(
    parent: &mut Parent,
    child_name: vector<u8>,
): DOFChild {
    dof::remove<vector<u8>, DOFChild>(&mut parent.id, child_name)
}

// Deletes a DOFChild given its name and parent object's mutable reference
public fun delete_dofchild(parent: &mut Parent, child_name: vector<u8>) {
    let DOFChild { id, .. } = remove_dofchild(parent, child_name);
    id.delete();
}

#[lint_allow(self_transfer)]
// Removes a DOFChild from the parent object and transfer it to the caller
public fun reclaim_dofchild(
    parent: &mut Parent,
    child_name: vector<u8>,
    ctx: &mut TxContext,
) {
    let child = remove_dofchild(parent, child_name);
    transfer::public_transfer(child, ctx.sender());
}
```

動的オブジェクトフィールドの場合、別のオブジェクトへの添付を削除した後、削除または転送できることに注意してください。動的オブジェクトフィールドは Sui オブジェクトだからです。しかし、動的フィールドは `key` アビリティを持たず、Sui オブジェクトではないため、同じことはできません。

## 動的フィールド vs. 動的オブジェクトフィールド

動的フィールドと動的オブジェクトフィールドをいつ使用すべきでしょうか？一般的に言えば、問題の子型が `key` アビリティを持つ場合は動的オブジェクトフィールドを使用し、そうでなければ動的フィールドを使用したいと思います。

根本的な理由の詳細な説明については、@sblackshear による[このフォーラム投稿](https://forums.sui.io/t/dynamicfield-vs-dynamicobjectfield-why-do-we-have-both/2095)を確認してください。

## `Table` の再検討

動的フィールドがどのように動作するかを理解したので、`Table` コレクションを動的フィールド操作の薄いラッパーと考えることができます。

演習として、Sui での Table タイプの[ソースコード](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/table.move)を確認し、以前に紹介した各操作が動的フィールド操作にどのようにマップされ、`Table` のサイズを追跡するための追加ロジックがあるかを確認できます。
