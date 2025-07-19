# オブジェクトラッピングの実例

成績証明書の例にオブジェクトラッピングの実装例を追加します。`WrappableTranscript` が `Folder` オブジェクトによってラップされ、`Folder` オブジェクトは意図されたアドレス/閲覧者によってのみアンパックでき、したがって内部の成績証明書にのみアクセスできるようにします。

## `WrappableTranscript` と `Folder` の変更

まず、前のセクションの 2 つのカスタムタイプ `WrappableTranscript` と `Folder` にいくつかの調整を行う必要があります。

1. `WrappableTranscript` の型定義に `key` アビリティを追加して、アセットとなり転送可能にする必要があります。

`key` と `store` のアビリティを持つカスタムタイプは、Sui Move ではアセットと見なされることを覚えておいてください。

```move
public struct WrappableTranscript has key, store {
    id: UID,
    history: u8,
    math: u8,
    literature: u8,
}
```

2. ラップされた成績証明書の意図された閲覧者のアドレスを示す追加フィールド `intended_address` を `Folder` 構造体に追加する必要があります。

```move
public struct Folder has key {
    id: UID,
    transcript: WrappableTranscript,
    intended_address: address
}
```

## 成績証明書リクエストメソッド

```move
public fun request_transcript(transcript: WrappableTranscript, intended_address: address, ctx: &mut TxContext){
    let folder_object = Folder {
        id: object::new(ctx),
        transcript,
        intended_address
    };
    //ラップされた成績証明書オブジェクトを意図されたアドレスに直接転送します
    transfer::transfer(folder_object, intended_address)
}
```

このメソッドは単純に `WrappableTranscript` オブジェクトを受け取り、`Folder` オブジェクトでラップして、ラップされた成績証明書を成績証明書の意図されたアドレスに転送します。

## 成績証明書アンラップメソッド

```move
public fun unpack_wrapped_transcript(folder: Folder, ctx: &mut TxContext) {
    // 成績証明書をアンパックする人が意図された閲覧者であることを確認
    assert!(folder.intended_address == ctx.sender(), ENotIntendedAddress);
    let Folder { id, transcript, .. } = folder;
    transfer::transfer(transcript, ctx.sender());
    id.delete();
}
```

このメソッドは、メソッド呼び出し元が成績証明書の意図された閲覧者である場合、`Folder` ラッパーオブジェクトから `WrappableTranscript` オブジェクトをアンラップし、メソッド呼び出し元に送信します。

_クイズ：なぜここでラッパーオブジェクトを手動で削除する必要があるのでしょうか？削除しないとどうなりますか？_

### Assert

`assert!` 構文を使用して、成績証明書をアンパックするトランザクションを送信するアドレスが、`Folder` ラッパーオブジェクトの `intended_address` フィールドと同じであることを確認しました。

`assert!` マクロは以下の形式の 2 つのパラメータを受け取ります：

```
assert!(<bool expression>, <code>)
```

ここで、ブール式 (boolean expression) は true と評価される必要があり、そうでなければエラーコード `<code>` で中断します。

### カスタムエラー (Custom Error)

上記では、エラーコードにデフォルトの 0 を使用していますが、以下の方法でカスタムエラー定数を定義することもできます：

```move
const ENotIntendedAddress: u64 = 1;
```

このエラーコードは、アプリケーションレベルで使用され、適切に処理できます。

**これまでに書いた内容の 2 番目の作業中バージョンはこちらです：[WIP transcript.move](../example_projects/transcript/sources/transcript_2.move_wip)**
