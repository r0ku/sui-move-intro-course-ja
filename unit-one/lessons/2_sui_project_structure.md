# Sui プロジェクト構造

## Sui モジュール (Module) とパッケージ (Package)

- Sui モジュールは、開発者が特定のアドレスの下で公開する関数 (function) と型 (type) をまとめたセットです

- Sui 標準ライブラリは `0x2` アドレスの下で公開されており、ユーザーがデプロイしたモジュールは Sui Move VM によって割り当てられた疑似ランダムアドレスの下で公開されます

- モジュールは `module` キーワードで始まり、その後にモジュール名と中括弧が続きます。中括弧の内部にモジュールの内容が配置されます：

  ```move
  module hello_world::hello_world;
  // module contents
  ```

- 公開されたモジュールは Sui では不変オブジェクト (immutable object) です。不変オブジェクトとは、変更、転送、削除が決してできないオブジェクトのことです。この不変性により、オブジェクトは誰にも所有されず、したがって誰でも使用できます

- Move パッケージは、Move.toml という名前のマニフェストファイル (manifest file) を持つモジュールの集合です

## Sui Move パッケージの初期化

以下の Sui CLI コマンドを使用して Sui パッケージのスケルトンを開始してください：

`sui move new <PACKAGE NAME>`

このユニットの例では、Hello World プロジェクトを開始します：

`sui move new hello_world`

これにより以下が作成されます：

- プロジェクトルートフォルダ `hello_world`
- パッケージに関するメタデータを含む `Move.toml` マニフェストファイル
- Sui Move スマートコントラクト (smart contract) のソースファイルを含む `sources/` サブフォルダ
- パッケージのテスト (test) を含む `tests/` サブフォルダ。tests ディレクトリに配置されたコードはオンチェーン (on-chain) で公開されず、テストでのみ利用可能です

### `Move.toml` マニフェスト構造

`Move.toml` はパッケージのマニフェストファイルで、プロジェクトルートフォルダに自動生成されます。

`Move.toml` はいくつかのセクションで構成されます：

- `[package]` パッケージを name（パッケージのインポート時に使用）、version（リリース管理用）、edition（Move 言語エディション、現在は 2024）などのフィールドで記述
- `[dependencies]` プロジェクトの依存関係 (dependency) を指定。各依存関係は git リポジトリまたはローカルディレクトリパスにできます。パッケージは依存関係からアドレスもインポートします
- `[dev-dependencies]` 開発とテストモードで依存関係をオーバーライド (override) するために使用。例えば、開発モードで異なるバージョンの Sui パッケージを使用したい場合、[dev-dependencies]セクションにカスタム依存関係仕様を追加できます
- `[addresses]` コード内でフルアドレスのショートカットとして使用できるアドレスのエイリアス (alias) を追加
- `[dev-addresses]` テストと開発モードでのみ既存のアドレスエイリアスをオーバーライドできます。新しいエイリアスは導入できず、既存のものだけをオーバーライドできます

#### サンプル `Move.toml` ファイル

これは、パッケージ名 `hello_world` で Sui CLI によって生成された `Move.toml` です：

```toml
[package]
name = "hello_world"
edition = "2024.beta" # edition = "legacy" to use legacy (pre-2024) Move
# license = ""           # e.g., "MIT", "GPL", "Apache 2.0"
# authors = ["..."]      # e.g., ["Joe Smith (joesmith@noemail.com)", "John Snow (johnsnow@noemail.com)"]

[dependencies]
# For remote import, use the `{ git = "...", subdir = "...", rev = "..." }`.
# Revision can be a branch, a tag, and a commit hash.
# MyRemotePackage = { git = "https://some.remote/host.git", subdir = "remote/path", rev = "main" }

# For local dependencies use `local = path`. Path is relative to the package root
# Local = { local = "../path/to" }

# To resolve a version conflict and force a specific version for dependency
# override use `override = true`
# Override = { local = "../conflicting/version", override = true }

[addresses]
hello_world = "0x0"

# Named addresses will be accessible in Move as `@name`. They're also exported:
# for example, `std = "0x1"` is exported by the Standard Library.
# alice = "0xA11CE"

[dev-dependencies]
# The dev-dependencies section allows overriding dependencies for `--test` and
# `--dev` modes. You can introduce test-only dependencies here.
# Local = { local = "../path/to/dev-build" }

[dev-addresses]
# The dev-addresses section allows overwriting named addresses for the `--test`
# and `--dev` modes.
# alice = "0xB0B"
```

## 依存関係 (Dependencies)

`[dependencies]` セクションは、プロジェクトの依存関係を指定するために使用されます。各依存関係はキーと値のペアとして指定され、キーは依存関係の名前、値は依存関係の仕様です。依存関係の仕様は、git リポジトリの URL またはローカルディレクトリへのパスにできます。

```toml
# git repository
Example = { git = "https://github.com/example/example.git", subdir = "path/to/package", rev = "framework/testnet" }

# local directory
MyPackage = { local = "../my-package" }
```

パッケージは他のパッケージからもアドレスをインポートします。例えば、Sui 依存関係はプロジェクトに `std` と `sui` アドレスを追加します。これらのアドレスは、コード内でアドレスのエイリアスとして使用できます。

Sui CLI のバージョン 1.45 以降、Sui システムパッケージ（`std`、`sui`、`system`、`bridge`、`deepbook`）は、明示的にリストされていない場合、自動的に依存関係として追加されます。

### オーバーライドによるバージョン競合の解決

依存関係が同じパッケージの競合するバージョンを持つことがあります。例えば、異なるバージョンの Example パッケージを使用する 2 つの依存関係がある場合、`[dependencies]` セクションで依存関係をオーバーライドできます。これを行うには、依存関係に `override` フィールドを追加します。`[dependencies]` セクションで指定された依存関係のバージョンが、依存関係自体で指定されたものの代わりに使用されます。

```toml
[dependencies]
Example = { override = true, git = "https://github.com/example/example.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/testnet" }
```

## Sui モジュールとパッケージの命名

- Sui Move モジュールとパッケージの命名規則では、スネークケース (snake casing) を使用します。つまり、this_is_snake_casing のような形式です。

- Sui モジュール名は、Rust パス区切り文字 `::` を使用してパッケージ名とモジュール名を分けます。例：

  1. `unit_one::hello_world` - `unit_one` パッケージ内の `hello_world` モジュール
  2. `capy::capy` - `capy` パッケージ内の `capy` モジュール

- Move の命名規則の詳細については、[Move ブックのスタイルセクション](https://move-language.github.io/move/coding-conventions.html#naming)を確認してください。
