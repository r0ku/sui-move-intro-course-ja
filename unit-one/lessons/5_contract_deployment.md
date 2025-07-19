# コントラクトデプロイメントと Hello World デモ

## 完全な Hello World サンプルプロジェクト

完全な Hello World プロジェクトは[このディレクトリ](../example_projects/hello_world)で確認できます。

## コントラクトのデプロイ (Deploy)

Sui CLI を使用してパッケージを Sui ネットワークにデプロイします。Sui の devnet、testnet、またはローカルノードのいずれにもデプロイできます。Sui CLI を該当のネットワークに設定し、ガス代を支払うのに十分なトークンを持っているだけです。

パッケージをデプロイするための Sui CLI コマンドは以下の通りです：

```bash
sui client publish [公開する必要があるパッケージへの絶対ファイルパス]
```

パッケージへの絶対ファイルパスが提供されない場合、`.` または現在のディレクトリがデフォルトになります。

コントラクトが正常にデプロイされた場合、出力は以下のようになります：

![Publish Output](../images/publish.png)

`Published Objects` セクションのオブジェクト ID は、今公開した Hello World パッケージのオブジェクト ID です。

これを変数にエクスポートしましょう。

```bash
export PACKAGE_ID=<前の出力からのパッケージオブジェクトID>
```

## トランザクションを通じたメソッド呼び出し

次に、今デプロイしたスマートコントラクトの `mint` 関数を呼び出して Hello World オブジェクトをミントしたいと思います。

`mint` がエントリー関数 (entry function) であるため、これができることに注意してください。

Sui CLI を使用したコマンドは以下の通りです：

```bash
sui client call --function mint --module hello_world --package $PACKAGE_ID
```

`mint` 関数が正常に呼び出され、Hello World オブジェクトが作成・転送された場合、コンソール出力は以下のようになります：

![Mint Output](../images/mint.png)

出力の `Created Objects` セクションのオブジェクト ID は、Hello World オブジェクトの ID です。

## Sui Explorer でのオブジェクト表示

[Sui Explorer](https://suiexplorer.com/)を使用して、今作成・転送した Hello World オブジェクトを表示しましょう。

右上のドロップダウンメニューから使用しているネットワークを選択してください。

ローカル開発ノードを使用している場合は、`Custom RPC URL` オプションを選択して以下を入力してください：

```bash
http://127.0.0.1:9000
```

前のトランザクションの出力からオブジェクト ID を検索すると、エクスプローラーでオブジェクトを見つけることができます：

![Explorer Output](../images/explorer.png)

オブジェクトのプロパティの下に「Hello World!」というテキストが表示されます。

お疲れさまでした。これでコースの最初のユニットが完了です。
