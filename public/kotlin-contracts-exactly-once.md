---
title: Kotlin Contracts の InvocationKind.EXACTLY_ONCE を「1 回呼び出す」と誤解してはいけない
tags:
  - Kotlin
  - KotlinContracts
private: false
updated_at: '2026-01-14T21:31:48+09:00'
id: e308b9e87fe69490d47d
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Kotlin には Contracts という、関数の内部動作に関するヒントをコンパイラに提供する機能があります。これにより、コンパイラはスマートキャストや変数の初期化判定をより賢く行えるようになります。

通常、コンパイラは関数の内部の実装まで深く解析しませんが、`contract` を使うことで「この関数は引数が `null` でない場合のみ `return` する」「この関数はラムダ式を必ず呼び出す」といった情報をコンパイラと契約（約束）できます。

本記事では、`InvocationKind.EXACTLY_ONCE` に着目して解説します。

## Kotlin Contracts の利用例

### スマートキャスト

例えば、`requireNotNull` は `contract` を利用しています。

`requireNotNull` は「正常に戻り値を返したならば、引数は `null` ではない：`returns() implies (value != null)`」とコンパイラに伝えています。もしこれがなければ、コンパイラは `null` かもしれないと判断し、以下のコードはコンパイルエラーになります。

```kotlin
val text: String? = getTextOrNull()
requireNotNull(text)

// ここで text は String (非 null) として扱える
println(text.length)
```

### 変数の確定的初期化

他にも、`run` も `contract` を利用しています。

通常、関数に渡すラムダ式の中で変数の初期化を試みても、コンパイラは「そのラムダ式が本当に 1 回だけ呼び出されるかわからない」ため、エラーとなります。しかし、`run` は「ラムダ式を必ず 1 回だけ呼び出す[^exactly-once]：`callsInPlace(block, InvocationKind.EXACTLY_ONCE)`」とコンパイラに伝えているため、ラムダ内での初期化を認めます。

[^exactly-once]: 正確には「1 回呼び出され、かつ正常終了する」という意味です。詳細は後述します。

```kotlin
val x: Int
run {
    // 初期化できる
    x = 100
}

// ここで x は初期化済みとみなされる
println(x)
```

## `runCatching` と `InvocationKind.EXACTLY_ONCE`

ここからが本題です。

「ラムダ式を 1 回だけ呼び出す」関数であれば、`InvocationKind.EXACTLY_ONCE` を指定してよいのでしょうか？

例として、例外を捕捉して `Result` 型を返す自作の `runCatching` を考えます。`block()` の呼び出し自体は確かに 1 回だけです。しかし、この関数に `EXACTLY_ONCE` を指定すると、`contract` と実際の制御フローに矛盾が生じてしまいます。

```kotlin
inline fun <R> myRunCatching(block: () -> R): Result<R> {
    contract {
        // 「block は必ず 1 回だけ呼ばれる」と考えてこれを指定する
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }

    return try {
        Result.success(block())
    } catch (e: Throwable) {
        Result.failure(e)
    }
}
```

### 発生する問題

次のコードを考えます。ラムダの途中で例外が発生するため `x` は初期化されません。しかし `myRunCatching` 自体は例外を catch して処理を続行します。コンパイラは `EXACTLY_ONCE` を信じて `x` を初期化済みと判断します。

結果として、未初期化の変数にアクセスするコードがコンパイルを通過し、実行時エラーや予期せぬ値の参照につながります。

```kotlin
val x: Int
myRunCatching {
    throwableFunction() // ここで例外が発生する
    x = 100 // 初期化が実行されない
}

// block の例外は catch されるため、以降の処理が続行される
println(x) // x が未初期化のままアクセスされる
```

### 本質的な誤解

`InvocationKind.EXACTLY_ONCE` に関する誤解は、次のように整理できます。

- 誤解：`block()` が 1 回だけ呼び出される
- 正解：`block()` が 1 回だけ呼び出され、かつ例外などで中断されず正常終了する

`run` の場合、ラムダ内で例外が発生すればその例外は外に伝播し、後続の処理は実行されません。そのため「初期化が完了していない状態で先に進む」ことが起きず、`EXACTLY_ONCE` を満たします。

一方、`runCatching` のようにラムダ内の例外を catch して処理を継続する関数では、`EXACTLY_ONCE` を満たせません。この場合、`EXACTLY_ONCE` ではなく `AT_MOST_ONCE` にするのが妥当でしょう。

## IDE による検出

幸いなことに、最近の IDE やコンパイラはこの矛盾を検知できるようになっています。Android Studio や IntelliJ IDEA で K2 Compiler を有効にしている場合、次のような警告が表示されます。

> Wrong invocation kind 'EXACTLY_ONCE' for 'block: () -> R' specified, the actual invocation kind is 'AT_MOST_ONCE'.

これは「ラムダは 1 回まで呼ばれる」という、より正確な解析結果を示しています。

## まとめ

- `InvocationKind.EXACTLY_ONCE` は「1 回呼び出される」ではなく、「1 回呼び出され、正常終了する」ことを意味する
- 例外を catch して処理を続行する関数では、`InvocationKind.EXACTLY_ONCE` を満たさない
- Kotlin Contracts は強力だが、コンパイラを誤誘導することも可能になるため、注意が必要

## 参考文献

1. [kotlin.contracts | Core API – Kotlin Programming Language](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.contracts/)
2. [InvocationKind | Core API – Kotlin Programming Language](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.contracts/-invocation-kind/)
3. [KEEP/proposals/KEEP-0139-kotlin-contracts.md at main · Kotlin/KEEP](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0139-kotlin-contracts.md)
4. [Fixing Indeterminate Behavior Caused by `InvocationKind.EXACTLY_ONCE` by Daiji256 · Pull Request #106 · michaelbull/kotlin-result](https://github.com/michaelbull/kotlin-result/pull/106)
