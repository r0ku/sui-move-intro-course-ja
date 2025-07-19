# Sui オブジェクトの操作

## イントロダクション

Sui Move は完全にオブジェクト中心の言語です。Sui 上のトランザクション (transaction) は、入力と出力の両方がオブジェクトである操作として表現されます。[ユニット 1、レッスン 4](../../unit-one/lessons/4_custom_types_and_abilities.md#custome-types-and-abilities)でこの概念に簡単に触れたように、Sui オブジェクト (Sui object) は Sui におけるストレージの基本単位です。すべては `struct` キーワードから始まります。

まず、学生の成績を記録する成績証明書を表す例から始めましょう：

```move
public struct Transcript {
    history: u8,
    math: u8,
    literature: u8,
}
```

上記の定義は通常の Move 構造体 (struct) ですが、Sui オブジェクトではありません。カスタム Move 型をグローバルストレージ (global storage) の Sui オブジェクトとしてインスタンス化するには、`key` アビリティと、構造体定義内にグローバルに一意な `id: UID` フィールドを追加する必要があります。

```move
public struct TranscriptObject has key {
    id: UID,
    history: u8,
    math: u8,
    literature: u8,
}
```

## Sui オブジェクトの作成

Sui オブジェクトの作成には一意な ID が必要です。現在の `TxContext` を渡して `sui::object::new` 関数を使用して新しい ID を作成します。

Sui では、すべてのオブジェクトは所有者 (owner) を持つ必要があり、それはアドレス、別のオブジェクト、または「共有 (shared)」のいずれかです。この例では、新しい `transcriptObject` をトランザクション送信者が所有するように決定しました。これは、Sui フレームワークの `transfer` 関数を使用し、`ctx.sender()` 関数を使用して現在のエントリー呼び出しの送信者のアドレスを取得することで実行されます。

オブジェクトの所有権 (ownership) については、次のセクションでより詳しく説明します。

```move
public fun create_transcript_object(history: u8, math: u8, literature: u8, ctx: &mut TxContext) {
  let transcript_object = TranscriptObject {
    id: object::new(ctx),
    history,
    math,
    literature,
  };
  transfer::transfer(transcript_object, ctx.sender())
}
```

_💡 注意：提供されたサンプルコードは警告メッセージを生成します：warning[Lint W01001]: non-composable transfer to sender。詳細については、記事["Sui Linters and Warnings Update Increases Coder Velocity"](https://blog.sui.io/linter-compile-warnings-update/)を参照してください。_

_💡 注意：Move はフィールドパニング (field punning) をサポートしており、フィールド名がバインドされた値変数の名前と同じ場合に、フィールド値をスキップできます。_
