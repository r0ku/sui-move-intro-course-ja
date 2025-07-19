# カスタムタイプとアビリティ (Custom Types and Abilities)

このセクションでは、Hello World サンプルコントラクトを段階的に作成し、カスタムタイプ (custom type) やアビリティ (ability) など、登場する Sui Move の基本概念を説明していきます。

## パッケージの初期化

（前のセクションをスキップした場合）[Sui バイナリのインストール](./1_set_up_environment.md#install-sui-binaries-locally)後、コマンドラインで以下のコマンドを使用して Hello World Sui パッケージを初期化できます：

`sui move new hello_world`

## コントラクトソースファイルの作成

お好みのエディターを使用して、`sources` サブフォルダ内に `hello.move` という名前の Move スマートコントラクトソースファイルを作成してください。

そして、以下のように空のモジュールを作成します：

```move
module hello_world::hello_world;
// module contents
```

### インポート文 (Import Statement)

Move では、アドレスによってモジュールを直接インポートできますが、コードを読みやすくするために、`use` キーワードでインポートを整理できます。

```move
use <Address/Alias>::<ModuleName>;
```

この例では、以下のモジュールをインポートする必要があります：

```move
use std::string;
```

### 暗黙的インポート (Implicit Import)

一部のモジュールは暗黙的にインポートされ、明示的な `use` インポートなしでモジュール内で利用できます。標準ライブラリ (Standard Library) では、これらのモジュールと型が含まれます：

- `std::vector`
- `std::option`
- `std::option::Option`

標準ライブラリと同様に、Sui フレームワークでも一部のモジュールと型が暗黙的にインポートされます。これは明示的な `use` インポートなしで利用できるモジュールと型のリストです：

- `sui::object`
- `sui::object::ID`
- `sui::object::UID`
- `sui::tx_context`
- `sui::tx_context::TxContext`
- `sui::transfer`

### メソッド呼び出し構文 (Method Call Syntax)

Move は、構文の利便性として `.` 演算子を使用したメソッド呼び出し構文をサポートしています。`.` の左側の値が関数の最初の引数になります。すべてのメソッド呼び出しはコンパイル時に静的に決定されます。

```move
// 従来の関数呼び出し構文
let sender = tx_context::sender(ctx);
let text = string::utf8(b"Hello World!");

// メソッド呼び出し構文
let sender = ctx.sender();
let text = b"Hello World!".to_string();
```

メソッド呼び出し構文は、関数の最初のパラメータがドットの前の型と一致する場合に機能します。コンパイラーは定義モジュール内の関数に対して自動的にメソッドエイリアスを作成し、必要に応じて自動的にレシーバーを借用します。

## カスタムタイプ (Custom Type)

Sui Move での構造体 (structure) は、キーと値のペアを含むカスタムタイプです。キーはプロパティの名前、値は格納されるものです。`struct` キーワードを使用して定義され、構造体は最大 4 つのアビリティを持つことができます。

### アビリティ (Ability)

アビリティは、コンパイラーレベルで型の動作を定義する Sui Move のキーワードです。

アビリティは、言語レベルで Sui Move におけるオブジェクトの動作を定義するために重要です。Sui Move でのアビリティの各固有の組み合わせは、それ自体がデザインパターンです。このコース全体を通して、アビリティと Sui Move での使用方法を学習します。

現在のところ、Sui Move には 4 つのアビリティがあることを知っておいてください：

- **copy**: 値をコピー（または値によってクローン）できる
- **drop**: スコープの終わりで値をドロップできる
- **key**: 値をグローバルストレージ操作のキーとして使用できる
- **store**: 値をグローバルストレージの構造体内に保持できる

#### アセット (Asset)

`key` と `store` のアビリティを持つカスタムタイプは、Sui Move では**アセット**と見なされます。アセットはグローバルストレージに格納され、アカウント間で転送できます。

### Hello World カスタムタイプ

Hello World の例では、オブジェクトを以下のように定義します：

```move
/// An object that contains an arbitrary string
public struct HelloWorldObject has key, store {
  	id: UID,
  	/// A string contained in the object
  	text: string::String
}
```

ここでの UID は、オブジェクトのグローバルに一意な ID を定義する Sui フレームワークタイプ (sui::object::UID) です。`key` アビリティを持つカスタムタイプには、ID フィールドが必要です。
