# ウィットネスデザインパターン (Witness Design Pattern)

次に、Sui Move でファンジブルトークンがどのように実装されているかを詳しく理解するために、ウィットネスパターンを理解する必要があります。

ウィットネス (witness) は、問題のリソースまたは型 `A` が、一時的な `witness` リソースが消費された後に一度だけ開始できることを証明するために使用されるデザインパターンです。`witness` リソースは使用後すぐに消費またはドロップされなければならず、`A` の複数のインスタンスを作成するために再利用できないことを保証します。

## ウィットネスパターンの例

以下の例では、`witness` リソースは `PEACE` であり、インスタンス化を制御したい型 `A` は `Guardian` です。

`witness` リソースタイプは、このリソースが関数に渡された後にドロップできるように `drop` キーワードを持つ必要があります。`PEACE` リソースのインスタンスが `create_guardian` メソッドに渡され、ドロップされる（`witness` の前のアンダースコアに注意）ことがわかり、`Guardian` のインスタンスを 1 つだけ作成できることを保証しています。

```move
/// Module that defines a generic type `Guardian<T>` which can only be
/// instantiated with a witness.
module witness::peace;

/// Phantom parameter T can only be initialized in the `create_guardian`
/// function. But the types passed here must have `drop`.
public struct Guardian<phantom T: drop> has key, store {
    id: UID
}

/// This type is the witness resource and is intended to be used only once.
public struct PEACE has drop {}

/// The first argument of this function is an actual instance of the
/// type T with `drop` ability. It is dropped as soon as received.
public fun create_guardian<T: drop>(_: T, ctx: &mut TxContext): Guardian<T> {
    Guardian { id: object::new(ctx) }
}

/// Module initializer is the best way to ensure that the
/// code is called only once. With `Witness` pattern it is
/// often the best practice.
fun init(witness: PEACE, ctx: &mut TxContext) {
    transfer::public_transfer(create_guardian(witness, ctx), ctx.sender())
}
```

_上記の例は、[Damir Shamanaev](https://github.com/damirka)による優れた書籍[Sui Move by Example](https://examples.sui.io/patterns/witness.html)から変更されたものです。_

### `phantom` キーワード

上記の例では、`Guardian` 型に `key` と `store` のアビリティを持たせ、アセットとして転送可能でグローバルストレージに永続化されるようにしたいと考えています。

また、`witness` リソース `PEACE` を `Guardian` に渡したいのですが、`PEACE` は `drop` アビリティのみを持っています。[アビリティ制約](./2_intro_to_generics.md#ability-constraints)と内部型に関する以前の議論を思い出すと、外部型 `Guardian` が持っているため、`PEACE` も `key` と `storage` を持つべきであるというルールが示唆されます。しかし、この場合、不要なアビリティを `witness` 型に追加したくありません。そうすることで望ましくない動作や脆弱性を引き起こす可能性があるためです。

この状況を回避するために `phantom` キーワードを使用できます。型パラメータが構造体定義内で使用されていないか、別の `phantom` 型パラメータの引数としてのみ使用されている場合、`phantom` キーワードを使用して Move タイプシステムに内部型のアビリティ制約ルールを緩和するよう求めることができます。`Guardian` がそのフィールドのいずれでも型 `T` を使用していないことがわかるので、`T` を `phantom` 型として安全に宣言できます。

`phantom` キーワードのより詳細な説明については、[Move Book の関連セクション](https://move-book.com/reference/generics#phantom-type-parameters)を確認してください。

## ワンタイムウィットネス (One Time Witness)

ワンタイムウィットネス (OTW: One Time Witness) は、ウィットネスパターンのサブパターンで、モジュールの `init` 関数を利用して `witness` リソースのインスタンスを 1 つだけ作成することを保証します（そのため型 `A` がシングルトンであることが保証されます）。

Sui Move では、型の定義が以下の特性を持つ場合、その型は OTW と見なされます：

- 型名がモジュール名を大文字にしたもの
- 型が `drop` アビリティのみを持つ

この型のインスタンスを取得するには、上記の例のようにモジュールの `init` 関数の最初の引数として追加する必要があります。Sui ランタイムは、モジュール公開時に OTW 構造体を自動的に生成します。

上記の例では、ワンタイムウィットネスデザインパターンを使用して `Guardian` がシングルトンであることを保証しています。
