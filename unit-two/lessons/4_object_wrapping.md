# オブジェクトラッピング (Object Wrapping)

Sui Move でオブジェクトを別のオブジェクト内にネストする方法は複数あります。最初に紹介する方法は、オブジェクトラッピング (object wrapping) と呼ばれます。

成績証明書の例を続けましょう。新しい `WrappableTranscript` 型と、関連するラッパー型 `Folder` を定義します。

```move
public struct WrappableTranscript has store {
    history: u8,
    math: u8,
    literature: u8,
}

public struct Folder has key {
    id: UID,
    transcript: WrappableTranscript,
}
```

上記の例では、`Folder` が `WrappableTranscript` をラップし、`Folder` は `key` アビリティを持つため、その ID を通じてアドレス可能です。

## オブジェクトラッピングの特性

構造体タイプが一般的に `key` アビリティを持つ Sui オブジェクト構造体に埋め込まれるためには、埋め込まれる構造体タイプは `store` アビリティを持つ必要があります。

オブジェクトがラップされると、ラップされたオブジェクトはオブジェクト ID を介して独立してアクセスできなくなります。代わりに、ラッパーオブジェクト自体の一部になります。さらに重要なことは、ラップされたオブジェクトはもはや Move 呼び出しで引数として渡すことができなくなり、唯一のアクセスポイントはラッパーオブジェクトを通してのみとなることです。

この特性により、オブジェクトラッピングは、特定のコントラクト呼び出し以外ではオブジェクトをアクセス不可能にする方法として使用できます。オブジェクトラッピングの詳細については、[こちら](https://docs.sui.io/devnet/build/programming-with-objects/ch4-object-wrapping)をご確認ください。
