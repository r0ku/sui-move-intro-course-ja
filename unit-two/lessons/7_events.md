# イベント (Events)

イベント (event) は、インデクサー (indexer) がオンチェーン上のアクションを追跡する主要な方法であるため、Sui Move スマートコントラクトにとって重要です。これをサーバーバックエンドでのログ記録、インデクサーをパーサーとして理解できます。

Sui のイベントもオブジェクトとして表されます。Sui には、Move イベント、公開イベント、オブジェクト転送イベントなど、いくつかのシステムレベルのイベントタイプがあります。システムイベントタイプの完全なリストについては、[こちらの Sui Events API ページ](https://docs.sui.io/build/event_api)を参照してください。

トランザクションのイベント詳細は、[Sui Explorer](https://suiexplorer.com/)の `Events` タブで確認できます：

![Event Tab](../images/eventstab.png)

## カスタムイベント (Custom Event)

開発者は Sui 上でカスタムイベントを定義することもできます。成績証明書がリクエストされた際をマークするカスタムイベントを以下の方法で定義できます。

```move
/// Event marking when a transcript has been requested
public struct TranscriptRequestEvent has copy, drop {
    // The Object ID of the transcript wrapper
    wrapper_id: ID,
    // The requester of the transcript
    requester: address,
    // The intended address of the transcript
    intended_address: address,
}
```

イベントを表すタイプは `copy` と `drop` のアビリティを持ちます。イベントオブジェクトはアセットを表すのではなく、含まれるデータにのみ関心があるため、複製可能で、スコープの終わりでドロップできます。

Sui でイベントを発行するには、[`sui::event::emit` メソッド](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/sui/event.md#function-emit)を使用するだけです。

このイベントを発行するように `request_transcript` メソッドを変更しましょう：

```move
public fun request_transcript(
    transcript: WrappableTranscript,
    intended_address: address,
    ctx: &mut TxContext,
) {
    let folder_object = Folder {
        id: object::new(ctx),
        transcript,
        intended_address,
    };
    event::emit(TranscriptRequestEvent {
        wrapper_id: folder_object.id.to_inner(),
        requester: ctx.sender(),
        intended_address,
    });
    // e transfer the wrapped transcript object directly to the intended address
    transfer::transfer(folder_object, intended_address);
}
```

Sui Explorer では、発行されたイベントが以下のように表示され、`TranscriptRequestEvent` イベントで定義した 3 つのデータフィールドが表示されます：

![Custom Event](../images/customevent.png)

**成績証明書サンプルプロジェクトの完全版はこちらです：[transcript.move](../example_projects/transcript/sources/transcript.move)**

Sui CLI クライアントと Sui Explorer を使用して、成績証明書の作成、リクエスト、アンパックを試して結果を確認してみてください。

これでユニット 2 は終了です。お疲れさまでした！
