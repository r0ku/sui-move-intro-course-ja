# パラメータ渡しとオブジェクト削除

## パラメータ渡し（`value`、`ref`、`mut ref`による）

Rust 言語に精通している方は、おそらく Rust の所有権システム (ownership system) についてご存知でしょう。Solidity と比較した movelang の利点の一つは、関数の相互作用に使用するアセット (asset) に対して、関数呼び出しが何をする可能性があるかを感覚的に把握できることです。以下にいくつかの例を示します：

```move
// スコアを取得することはできますが、変更はできません
public fun view_score(transcript_object: &TranscriptObject): u8{
    transcript_object.literature
}

// スコアを表示および編集することはできますが、削除はできません
public fun update_score(transcript_object: &mut TranscriptObject, score: u8){
    transcript_object.literature = score
}

// スコアに対して、表示、編集、成績証明書全体の削除を含む、あらゆる操作が可能です
public fun delete_transcript(transcript_object: TranscriptObject){
    let TranscriptObject {id, .. } = transcript_object;
    id.delete();
}
```

## オブジェクト削除と構造体アンパック (Struct Unpacking)

上記の例の `delete_transcript` メソッドは、Sui でオブジェクトを削除する方法を示しています。

1. オブジェクトを削除するには、まずオブジェクトをアンパック (unpack) してそのオブジェクト ID を取得する必要があります。アンパックは、Move の特権構造体操作ルール (privileged struct operation rule) により、オブジェクトを定義するモジュール内でのみ実行できます：

- 構造体タイプは、構造体を定義するモジュール内でのみ作成（「パック」）、破棄（「アンパック」）できます
- 構造体のフィールドは、構造体を定義するモジュール内でのみアクセス可能です

これらのルールに従って、定義モジュールの外で構造体を変更したい場合は、これらの操作のためのパブリックメソッド (public method) を提供する必要があります。

2. 構造体をアンパックして ID を取得した後、オブジェクト ID で `id.delete()` フレームワークメソッドを呼び出すだけでオブジェクトを削除できます。

_💡 注意：上記のメソッドの `..`（ドットドット）は、構造体アンパックで残りのフィールドを無視することを示しています。これにより、必要なフィールドのみを抽出し、残りを無視できます。_

**これまでに書いた内容の作業中バージョンはこちらです：[WIP transcript.move](../example_projects/transcript/sources/transcript__1.move_wip)**
