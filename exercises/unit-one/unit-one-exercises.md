# Unit One 演習問題

## 問 1

このユニットでは、Move でアセットを定義する際に重要な[アビリティ (abilities)](../../unit-one/lessons/3_custom_types_and_abilities.md)を紹介しました。しかし、4 つのアビリティの一部の組み合わせは、アビリティの動作により、許可された場合に安全でない、または矛盾する動作を引き起こす可能性があるため、Move では不正です。

以下のアビリティの組み合わせを合法 (legal) または不正 (illegal) としてマークしてください：

1. `copy` + `drop`
2. `copy` + `key`
3. `copy` + `store`
4. `drop` + `key`
5. `drop` + `store`
6. `store` + `key`
7. `copy` + `drop` + `store`
8. `copy` + `drop` + `key`
9. `copy` + `key` + `store`
10. `drop` + `key` + `store`
11. `copy` + `drop` + `key` + `store`

不正な組み合わせのそれぞれについて、その組み合わせが許可された場合に発生する矛盾する動作を簡潔に説明してください。

_ヒント：コンパイラを使用してこれらの組み合わせをテストできます。_

## 問 2

`sui::transfer::transfer` と `sui::transfer::public_transfer` の違いは何ですか？
