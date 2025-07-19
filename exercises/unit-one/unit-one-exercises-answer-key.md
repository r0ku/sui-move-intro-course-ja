# Unit One 演習問題解答キー

## 問 1

1. `copy` + `drop`：合法
2. `copy` + `key`：不正、`copy` が `key` と矛盾
3. `copy` + `store`：合法
4. `drop` + `key`：不正、`drop` が `key` と矛盾
5. `drop` + `store`：合法
6. `store` + `key`：合法
7. `copy` + `drop` + `store`：合法
8. `copy` + `drop` + `key`：不正、`drop` が `key` と矛盾
9. `copy` + `key` + `store`：不正、`copy` が `key` と矛盾
10. `drop` + `key` + `store`：不正、`drop` が `key` と矛盾
11. `copy` + `drop` + `key` + `store`：不正、`drop` が `key` と矛盾

## 問 2

`transfer` はオブジェクトが `key` を持つことを要求し、`transfer` が呼び出されるモジュールと同じモジュールでオブジェクトが定義されている必要があります。

`public_transfer` は転送されるオブジェクトが `key` と `store` の両方のアビリティを持つことを要求しますが、オブジェクトが定義されているモジュールの外部からでも呼び出すことができます。
