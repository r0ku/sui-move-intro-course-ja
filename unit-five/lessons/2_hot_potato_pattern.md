# ホットポテトパターン (Hot Potato Pattern)

ホットポテト (hot potato) は能力 (capability) を持たない構造体で、そのモジュール内でのみパック・アンパックできます。ホットポテトパターンは PTB メカニクスを活用し、アプリケーションがトランザクション終了前にユーザーに決定されたビジネスロジックを満たすことを強制したい場合に一般的に使用されます。簡単に言えば、トランザクションコマンド A によってホットポテト値が返された場合、同じ PTB 内の後続のコマンド B でそれを消費する必要があります。ホットポテトパターンの最も人気のあるユースケースはフラッシュローン (flashloan) です。

## 型定義

```move
module flashloan::flashloan;

// === Imports ===
use sui::sui::SUI;
use sui::coin::{Self, Coin};
use sui::balance::{Self, Balance};
use sui::object::{UID};
use sui::tx_context::{TxContext};

/// For when the loan amount exceed the pool amount
const ELoanAmountExceedPool: u64 = 0;
/// For when the repay amount do not match the initial loan amount
const ERepayAmountInvalid: u64 = 1;

/// A "shared" loan pool.
/// For demonstration purpose, we assume the loan pool only allows SUI.
public struct LoanPool has key {
    id: UID,
    amount: Balance<SUI>,
}

/// A loan position.
/// This is a hot potato struct, it enforces the users
/// to repay the loan in the end of the transaction or within the same PTB.
public struct Loan {
    amount: u64,
}
```

ユーザーが借用するための資金保管庫として機能する `LoanPool` 共有オブジェクトがあります。簡単にするため、このプールは SUI のみを受け入れます。次に、ホットポテト構造体である `Loan` があり、これを使用してユーザーにトランザクション終了前にローンを返済することを強制します。`Loan` は借用額である `amount` フィールドを 1 つだけ持ちます。

## 借用 (Borrow)

```move
/// Function allows users to borrow from the loan pool.
/// It returns the borrowed [`Coin<SUI>`] and the [`Loan`] position
/// enforcing users to fulfill before the PTB ends.
public fun borrow(
    pool: &mut LoanPool,
    amount: u64,
    ctx: &mut TxContext,
): (Coin<SUI>, Loan) {
    assert!(amount <= pool.amount.value(), ELoanAmountExceedPool);
    (
        pool.amount.split(amount).into_coin(ctx),
        Loan {
            amount,
        },
    )
}
```

ユーザーは `borrow()` を呼び出すことで `LoanPool` からお金を借りることができます。基本的に、ユーザーが後続の関数呼び出しで自由に使用できる `Coin<SUI>` を返します。`Loan` ホットポテト値も返されます。前述のとおり、`Loan` を消費する唯一の方法は、同じモジュールの関数でアンパックすることです。これにより、アプリケーション自体のみがホットポテトを消費する方法を決定する権利を持ち、外部の当事者は持ちません。

## 返済 (Repay)

```move
/// Repay the loan
/// Users must execute this function to ensure the loan is repaid before the
/// transaction ends.
public fun repay(pool: &mut LoanPool, loan: Loan, payment: Coin<SUI>) {
    let Loan { amount } = loan;
    assert!(payment.value() == amount, ERepayAmountInvalid);

    pool.amount.join(payment.into_balance());
}
```

ユーザーは PTB 終了前のある時点でローンを `repay()` する必要があります。`Loan` をアンパックして消費します。そうしなければ、`Loan` は非 `drop` なので、直接アクセス `loan.amount` でそのフィールドを使用するとコンパイラエラーが発生します。アンパック後、ローン額を使用して有効な支払いチェックを実行し、それに応じて `LoanPool` を更新します。

## 例

SUI 額を借り、それを使用してダミー NFT をミントし、それを売却して債務を返済するフラッシュローンの例を作成してみましょう。Sui CLI で PTB を使用して、これらすべてを 1 つのトランザクションで実行する方法を学びます。

```move
/// A dummy NFT to represent the flashloan functionality
public struct NFT has key {
    id: UID,
    price: Balance<SUI>,
}

/// Mint NFT
public fun mint_nft(payment: Coin<SUI>, ctx: &mut TxContext): NFT {
    NFT {
        id: object::new(ctx),
        price: payment.into_balance(),
    }
}

/// Sell NFT
public fun sell_nft(nft: NFT, ctx: &mut TxContext): Coin<SUI> {
    let NFT { id, price } = nft;
    id.delete();
    price.into_coin(ctx)
}
```

以前のガイドを使用してスマートコントラクトを公開できるはずです。スマートデプロイメント後、パッケージ ID と共有 `LoanPool` オブジェクトが必要です。後で使用できるようにエクスポートしましょう。

```bash
export LOAN_PACKAGE_ID=<パッケージID>
export LOAN_POOL_ID=<ローンプールのオブジェクトID>
```

`flashloan::deposit_pool` 関数を使用して SUI 額をデポジットする必要があります。デモンストレーション目的で、ローンプールに 10_000 MIST をデポジットします。

```bash
sui client ptb \
--split-coins gas "[10000]" \
--assign coin \
--move-call $LOAN_PACKAGE_ID::flashloan::deposit_pool @$LOAN_POOL_ID coin.0 \

```

では、`borrow() -> mint_nft() -> sell_nft() -> repay()` という PTB を構築しましょう。

```bash
sui client ptb \
--move-call $LOAN_PACKAGE_ID::flashloan::borrow @$LOAN_POOL_ID 10000 \
--assign borrow_res \
--move-call $LOAN_PACKAGE_ID::flashloan::mint_nft borrow_res.0 \
--assign nft \
--move-call $LOAN_PACKAGE_ID::flashloan::sell_nft nft \
--assign repay_coin \
--move-call $LOAN_PACKAGE_ID::flashloan::repay @$LOAN_POOL_ID borrow_res.1 repay_coin \

```

_クイズ：PTB の最後で `repay()` を呼び出さない場合、何が起こりますか？自分で試してみてください。_

_💡 注意：詳細について PTB を検査するために、[SuiVision](https://testnet.suivision.xyz/)や[SuiScan](https://suiscan.xyz/testnet/home)をチェックすることをお勧めします。_
