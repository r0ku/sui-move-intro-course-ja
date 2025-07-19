# ジェネリック入門 (Intro to Generics)

ジェネリック (generic) は、具体的な型やその他のプロパティの抽象的な代替物です。[Rust のジェネリック](https://doc.rust-lang.org/stable/book/ch10-00-generics.html)と同様に動作し、Sui Move コードを書く際により大きな柔軟性を可能にし、ロジックの重複を避けるために使用できます。

ジェネリックは Sui Move の重要な概念であり、それらがどのように動作するかを理解し、直感を持つことが重要です。このセクションに時間をかけて、すべての部分を完全に理解してください。

## ジェネリックの使用

### 構造体でのジェネリックの使用

Sui Move で任意の型を保持できるコンテナ `Box` を作成するためにジェネリックを使用する基本的な例を見てみましょう。

まず、ジェネリックを使わずに、`u64` 型を保持する `Box` を以下のように定義できます：

```move
module generics::storage;

public struct Box {
    value: u64
}
```

しかし、この型は `u64` 型の値のみを保持できます。`Box` を任意のジェネリック型を保持できるようにするには、ジェネリックを使用する必要があります。コードは以下のように変更されます：

```move
module generics::storage;
public struct Box<T> {
    value: T
}
```

#### アビリティ制約 (Ability Constraint)

ジェネリックに渡される型が特定のアビリティを持つことを強制する条件を追加できます。構文は以下のようになります：

```move
module generics::storage;
// T must be copyable and droppable
public struct Box<T: store + drop> has key, store {
    value: T
}
```

💡 ここで重要なことは、上記の例の内部型 `T` は外部コンテナ型により特定のアビリティ制約を満たす必要があることです。この例では、`Box` が `store` と `key` を持つため、`T` は `store` を持つ必要があります。しかし、`T` はコンテナが持たないアビリティ（この例では `drop`）も持つことができます。

直感的には、コンテナが自分と同じルールに従わない型を含むことが許可されていれば、コンテナは自分自身のアビリティに違反することになります。内容も保存可能でなければ、ボックスがどうして保存可能になるでしょうか？

次のセクションでは、`phantom` という特別なキーワードを使用して、特定の場合にこのルールを回避する方法があることを見ていきます。

_💡 ジェネリック型の例については、`example_projects` の下の[generics プロジェクト](../example_projects/generics/)を参照してください。_

### 関数でのジェネリックの使用

`value` フィールドに任意の型のパラメータを受け入れることができる `Box` のインスタンスを返す関数を書くには、関数定義でもジェネリックを使用する必要があります。関数は以下のように定義できます：

```move
public fun create_box<T>(value: T): Box<T> {
    Box<T> { value }
}
```

関数が `value` に特定の型のみを受け入れるように制限したい場合は、以下のように関数シグネチャー (function signature) でその型を指定するだけです：

```move
public fun create_box(value: u64): Box<u64> {
    Box<u64>{ value }
}
```

これは、同じジェネリック `Box` 構造体を使用しながら、`create_box` メソッドに対して `u64` 型の入力のみを受け入れます。

#### ジェネリックを持つ関数の呼び出し

ジェネリックを含むシグネチャーを持つ関数を呼び出すには、以下の構文のように、角括弧内で型を指定する必要があります：

```move
// value will be of type storage::Box<bool>
let bool_box = storage::create_box<bool>(true);
// value will be of the type storage::Box<u64>
let u64_box = storage::create_box<u64>(1000000);
```

#### Sui CLI を使用したジェネリック関数の呼び出し

Sui CLI からシグネチャーにジェネリックを持つ関数を呼び出すには、`--type-args` フラグを使用して引数の型を定義する必要があります。

以下は、`0x2::sui::SUI` 型のコインを含むボックスを作成するために `create_box` 関数を呼び出す例です：

```bash
sui client call --package $PACKAGE --module $MODULE --function "create_box" --args $OBJECT_ID --type-args 0x2::sui::SUI
```

## 高度なジェネリック構文

複数のジェネリック型など、Sui Move でのジェネリックの使用に関するより高度な構文については、[Move Book のジェネリックに関する優れたセクション](https://move-book.com/reference/generics)を参照してください。

しかし、ファンジブルトークンに関する現在のレッスンでは、ジェネリックがどのように動作するかについて十分に知識があるので、先に進むことができます。
