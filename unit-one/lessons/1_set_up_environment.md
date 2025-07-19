# 開発環境のセットアップ

Sui Move 入門コースへようこそ。この最初のユニットでは、Sui Move での開発環境のセットアップ過程を説明し、Sui の世界への優しい入門として基本的な Hello World プロジェクトを作成します。

## Sui のインストール

Move はコンパイル言語なので、Move プログラムを書いて実行するためにはコンパイラーをインストールする必要があります。コンパイラーは Sui バイナリに含まれており、以下のいずれかの方法でインストールまたはダウンロードできます。

### suiup を使用したインストール（推奨）

Sui をインストールする最良の方法は `suiup` を使用することです。これにより、バイナリのインストールと、異なる環境（例：testnet と mainnet）用の異なるバージョンのバイナリの管理が簡単になります。

`suiup` のインストール手順は[リポジトリの README](https://github.com/MystenLabs/suiup)で確認できます。

Sui をインストールするには、以下のコマンドを実行してください：

```bash
suiup install sui
```

### バイナリのダウンロード

最新の Sui バイナリを[リリースページ](https://github.com/MystenLabs/sui/releases)からダウンロードできます。バイナリは macOS、Linux、Windows 向けに利用可能です。教育目的と開発では、mainnet バージョンの使用を推奨します。

### Homebrew を使用したインストール（macOS）

Homebrew パッケージマネージャーを使用して Sui をインストールできます。

```bash
brew install sui
```

### Chocolatey を使用したインストール（Windows）

Windows 向けの Chocolatey パッケージマネージャーを使用して Sui をインストールできます。

```bash
choco install sui
```

### Cargo を使用したビルド（macOS、Linux）

Cargo パッケージマネージャー（Rust が必要）を使用して Sui をローカルでインストール・ビルドできます。

```bash
cargo install --git https://github.com/MystenLabs/sui.git sui --branch mainnet
```

testnet や devnet を対象とする場合は、ここでブランチターゲットを `testnet` または `devnet` に変更してください。

以下のコマンドでシステムに最新の Rust バージョンがあることを確認してください。

```bash
rustup update stable
```

### インストールの確認

バイナリが正常にインストールされたか確認してください：

```bash
sui --version
```

sui バイナリが正常にインストールされていれば、ターミナルにバージョン番号が表示されます。

### トラブルシューティング

インストールプロセスのトラブルシューティングについては、[Sui インストールガイド](https://docs.sui.io/build/install)を参照してください。

## Sui バイナリがプリインストールされた Docker イメージの使用

1. [Docker をインストール](https://docs.docker.com/get-docker/)

2. Sui 公式 docker イメージをプル

   `docker pull mysten/sui-tools:devnet`

3. Docker コンテナを開始してシェルにアクセス：

   `docker run --name suidevcontainer -itd mysten/sui-tools:devnet`

   `docker exec -it suidevcontainer bash`

_💡 注意：上記の Docker イメージがお使いの CPU アーキテクチャと互換性がない場合は、お使いの CPU アーキテクチャに適したベース[Rust](https://hub.docker.com/_/rust) Docker イメージから始めて、上記の説明に従って Sui バイナリと前提条件をインストールできます。\_

## （オプション）Move Analyzer プラグイン (Plug-in) で VS Code を設定

1. VS Marketplace から[Move Analyzer プラグイン](https://marketplace.visualstudio.com/items?itemName=move.move-analyzer)をインストール

2. Sui スタイルのウォレットアドレス (wallet address) の互換性を追加：

   `cargo install --git https://github.com/move-language/move move-analyzer --features "address20"`

## Sui CLI 基本的な使用方法

[リファレンスページ](https://docs.sui.io/build/cli-client)

### 初期化

- `do you want to connect to a Sui Full node server?` に対して `Y` を入力し、`Enter`を押して Sui Devnet フルノード (full node) にデフォルト接続
- キースキーム (key scheme) 選択で `0` を入力して[`ed25519`](https://ed25519.cr.yp.to/)を選択

### ネットワーク管理

- ネットワーク切り替え: `sui client switch --env [network alias]`
- デフォルトネットワークエイリアス (alias):
  - localnet: http://0.0.0.0:9000
  - devnet: https://fullnode.devnet.sui.io:443
- 現在のすべてのネットワークエイリアスのリスト: `sui client envs`
- 新しいネットワークエイリアスの追加: `sui client new-env --alias <ALIAS> --rpc <RPC>`
  - 次のコマンドで testnet エイリアスの追加を試してください: `sui client new-env --alias testnet --rpc https://fullnode.testnet.sui.io:443`

### アクティブアドレスとガスオブジェクト (Gas Object) の確認

- キーストア (key store) の現在のアドレス確認: `sui client addresses`
- アクティブアドレス確認: `sui client active-address`
- 制御されているすべてのガスオブジェクトのリスト: `sui client gas`

## Devnet または Testnet Sui トークンの取得

1. [Sui Discord に参加](https://discord.gg/sui)
2. 認証ステップを完了
3. devnet トークン用の[`#devnet-faucet`](https://discord.com/channels/916379725201563759/971488439931392130)チャンネル、または testnet トークン用の[`#testnet-faucet`](https://discord.com/channels/916379725201563759/1037811694564560966)チャンネルに入る
4. `!faucet <WALLET ADDRESS>`と入力
