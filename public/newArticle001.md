---
title: newArticle001
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# Kotlin Contracts の InvocationKind.EXACTLY_ONCE を「1 回呼び出す」と誤解してはいけない

## はじめに

Kotlin には Contracts という、関数の内部動作に関するヒントをコンパイラに提供する機能があります。これにより、コンパイラはスマートキャストや変数の初期化判定をより賢く行えるようになります。

通常、コンパイラは関数の内部の実装まで深く解析しませんが、`contract` を使うことで「この関数は引数が `null` でない場合のみ `return` する」「この関数はラムダ式を必ず 1 回実行する」といった情報をコンパイラに約束（契約）できます。

## Kotlin Contracts の利用例

### スマートキャスト

例えば、`requireNotNull` は `contract` を利用しています。

`requireNotNull` は「正常に戻り値を返したならば、引数は `null` ではない：`returns() implies (value != null)`」と実装されているためです。もしこの契約がなければ、コンパイラは `null` かもしれないと判断し、コンパイルエラーになります。

```kotlin
val text: String? = getTextOrNull()
requireNotNull(text)

// ここで text は String (非 null) として扱える
println(text.length) 
```

### 変数の確定的初期化

他にも、`run` も `contract` を利用しています。

通常、関数に渡すラムダ式の中で変数を初期化を試みても、コンパイラは「そのラムダ式が本当に 1 度だけ実行されるかわからない」ため、エラーとなります。しかし、`run` は「ラムダ式を必ず 1 回実行する：`callsInPlace(block, InvocationKind.EXACTLY_ONCE)`」と実装されているため、コンパイラはラムダ内での初期化を認めます。

```kotlin
val x: Int
run {
    // 初期化できる
    x = 100
}

// ここで x は初期化済みとみなされる
println(x)
```

## `runCatching` と `EXACTLY_ONCE`

ここからが本題です。

さて、「ラムダ式を 1 回実行する」関数であれば、`EXACTLY_ONCE` をつければ良いのでしょうか？

例えば、例外を捕捉して `Result` 型を返す自作の `runCatching` を考えてみます。`block()` は確かに 1 回呼び出されるため、一見良さそうに見えます。しかし、この `contract` 定義は不具合を引き起こす可能性があります。

```kotlin
inline fun <R> myRunCatching(block: () -> R): Result<R> {
    contract {
        // 「block は必ず 1 回呼ばれる」と考えてこれを指定する
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }

    return try {
        Result.success(block())
    } catch (e: Throwable) {
        Result.failure(e)
    }
}
```

### 不具合

不具合は、以下のようなコードで発生します。

ラムダ内で例外が発生し、`x` が初期化されません。しかし、`myRunCatching` は例外を `catch` し、処理を続行します。ここで、コンパイラは `EXACTLY_ONCE`（必ず最後まで実行された）を信じて、`x` は初期化済みだと判断するため、コンパイルは通ります。

結果、実行時に未初期化の値にアクセスして、不定値を参照することになります。

```kotlin
val x: Int
myRunCatching {
    throwableFunction() // ここで例外が発生する
    x = 100 // 初期化が実行されない
}

// block の例外は catch されるため、以降の処理が続行される
println(x) // x が未初期化のままアクセスされる（不定値）
```

### 本質的な誤解

`EXACTLY_ONCE` の誤解は、以下のように整理できます。

- 誤解：`block()` が 1 度だけ呼び出される
- 正解：`block()` が 1 度だけ呼び出され、最後まで完走する

`run` の場合、ラムダ内で例外が発生すれば、その後のコードは実行されないため安全です。一方、`runCatching` のようにラムダが途中で失敗しても、正常終了して先に進む場合は、`EXACTLY_ONCE` の契約を満たせません。

## IDE の警告

幸いなことに、最近の IDE やコンパイラはこの矛盾を検知できます。Android Studio や IntelliJ IDEA で Kotlin の K2 モード（K2 Compiler）を有効にしている場合に、以下の警告が出ます。

> Wrong invocation kind 'EXACTLY_ONCE' for 'block: () -> R' specified, the actual invocation kind is 'AT_MOST_ONCE'.

## まとめ

Kotlin Contracts は強力ですが、「コンパイラを騙せてしまう」機能でもあります。

例外処理など複雑な処理が絡むときには特に注意が必要です。実装をよく理解し、テストを実施し、IDE の警告などをうまく活用して、安全で便利なコードを書きましょう。

## 参考文献

1. [kotlin.contracts | Core API – Kotlin Programming Language](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.contracts/)
2. [InvocationKind | Core API – Kotlin Programming Language](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.contracts/-invocation-kind/)
3. [KEEP/proposals/KEEP-0139-kotlin-contracts.md at main · Kotlin/KEEP](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0139-kotlin-contracts.md)
4. [Fixing Indeterminate Behavior Caused by `InvocationKind.EXACTLY_ONCE` by Daiji256 · Pull Request #106 · michaelbull/kotlin-result](https://github.com/michaelbull/kotlin-result/pull/106)
