# ケイパビリティデザインパターン (Capability Design Pattern)

これで成績証明書公開システムの基礎ができたので、スマートコントラクトにアクセス制御を追加したいと思います。

ケイパビリティ (capability) は、オブジェクト中心モデルを使用してきめ細かなアクセス制御を可能にする、Move でよく使用されるパターンです。このケイパビリティオブジェクトを定義する方法を見てみましょう：

```move
// Type that marks the capability to create, update, and delete transcripts
public struct TeacherCap has key {
    id: UID
}
```

成績証明書に対する特権アクション (privileged action) を実行するケイパビリティをマークする新しい構造体 `TeacherCap` を定義します。ケイパビリティを転送不可にしたい場合は、構造体に `store` アビリティを追加しないだけです。

\*💡 注意：これは、ソウルバウンドトークン (SBT: soulbound token) の同等物を Move で簡単に実装する方法でもあります。`key` アビリティを持つが `store` アビリティを持たない構造体を定義するだけです。

## ケイパビリティオブジェクトの受け渡しと消費

次に、`TeacherCap` ケイパビリティオブジェクトを持つ人が呼び出すべきメソッドを変更して、ケイパビリティを追加パラメータとして受け取り、すぐに消費するようにする必要があります。

例えば、`create_wrappable_transcript_object` メソッドでは、以下のように変更できます：

```move
public fun create_wrappable_transcript_object(
    _: &TeacherCap,
    history: u8,
    math: u8,
    literature: u8,
    ctx: &mut TxContext,
) {
    let wrappable_transcript = WrappableTranscript {
        id: object::new(ctx),
        history,
        math,
        literature,
    };
    transfer::public_transfer(wrappable_transcript, ctx.sender())
}
```

`TeacherCap` ケイパビリティオブジェクトへの参照 (reference) を渡し、未使用の変数とパラメータの `_` 記法ですぐに消費します。オブジェクトへの参照のみを渡しているため、参照の消費は元のオブジェクトに影響を与えないことに注意してください。

_クイズ：`TeacherCap` を値で渡そうとするとどうなりますか？_

これは、`TeacherCap` オブジェクトを持つアドレスのみがこのメソッドを呼び出すことができることを意味し、このメソッドでアクセス制御を効果的に実装しています。

成績証明書に対して特権アクションを実行するコントラクトの他のすべてのメソッドにも同様の変更を行います。

## 初期化関数 (Initializer Function)

モジュールの初期化関数 (initializer function) は、モジュールの公開時に一度だけ呼び出されます。これはスマートコントラクトの状態を初期化するのに便利で、初期のケイパビリティオブジェクトセットを送信するためによく使用されます。

この例では、`init` メソッドを以下のように定義できます：

```move
/// Module initializer is called only once on module publish.
fun init(ctx: &mut TxContext) {
    transfer::transfer(TeacherCap {
        id: object::new(ctx)
    }, ctx.sender())
}
```

これにより、`TeacherCap` オブジェクトのコピーが 1 つ作成され、モジュールが最初に公開されたときに公開者のアドレスに送信されます。

公開トランザクションの効果は、以下のように[Sui Explorer](../../unit-one/lessons/6_hello_world.md#viewing-the-object-with-sui-explorer)で確認できます：

![Publish Output](../images/publish.png)

上記のトランザクションから作成された 2 番目のオブジェクトは `TeacherCap` オブジェクトのインスタンスで、公開者アドレスに送信されます：

![Teacher Cap](../images/teachercap.png)

_クイズ：最初に作成されたオブジェクトは何でしたか？_

## 追加教師または管理者の追加

追加のアドレスに管理者アクセス権を与えるには、以下のように追加の `TeacherCap` オブジェクトを作成して送信するメソッドを定義するだけです：

```move
public fun add_additional_teacher(
    _: &TeacherCap,
    new_teacher_address: address,
    ctx: &mut TxContext,
) {
    transfer::transfer(
        TeacherCap {
            id: object::new(ctx),
        },
        new_teacher_address,
    )
}
```

このメソッドは `TeacherCap` を再利用してアクセスを制御しますが、必要に応じて、sudo アクセスを示す新しいケイパビリティ構造体を定義することもできます。

**これまでに書いた内容の 3 番目の作業中バージョンはこちらです：[WIP transcript.move](../example_projects/transcript/sources/transcript_3.move_wip)**
