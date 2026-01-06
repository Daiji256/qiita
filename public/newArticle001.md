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

# Kotlin Contracts の InvocationKind.EXACTLY_ONCE を勘違いしていた

## 3 行まとめ

- Kotlin Contracts はコンパイラとの契約であり、慎重に利用すべき
- `InvocationKind.EXACTLY_ONCE` は「1 回呼ばれる」ではなく「1 回、正常に完了した」である
- 最近の IDE は賢い

## Kotlin Contracts とは

Kotlin Contracts とは、Kotlin コンパイラに対して関数の動作に関する追加情報を提供するための機能です。これにより、コンパイラはコードの振る舞いをより正確に理解し、型安全性や最適化を向上させることができます。

Kotlin Contracts を使用するには、`ExperimentalContracts` をオプトインする必要があります。実験的かつコンパイラに強い影響を与える機能であるため、慎重に利用する必要があります。

### 例 1: `requireNotNull`

Kotlin Contracts が利用された身近の例として、`requireNotNull` があります。次の実装は問題なくコンパイルできます。これは `requireNotNull` が `x` が非 `null` なことを保証するため、以降のコードでは `x` を非 `null` として扱えるからです。そのため、`x.length` の呼び出しも問題ありません。

```kotlin
val x: String? = // ...
requireNotNull(x)

x.length
```

これが実現できているのは `requireNotNull` は `contract` により、コンパイラに `x` が非 `null` であることを伝えているためです。コンパイラは `contract` の指定を信用して、コンパイルを通します。

もし、`contract` がなければ “Only safe (?.) or non-null asserted (!!.) calls are allowed on a nullable receiver of type 'String?'.” とエラーが出てコンパイルできません。

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T : Any> requireNotNull(value: T?): T {
    contract {
        returns() implies (value != null)
    }
    // ...
}
```

### 例 2: `run`

`contract` を利用した関数のもう一つの例として、`run` 関数があります。`run` 関数はレシーバーオブジェクトに対してラムダ式を適用し、その結果を返します。

`run` では `InvocationKind.EXACTLY_ONCE` が指定されています。これは「`block` が正確に 1 回呼ばれる」ことを意味します。

```kotlin
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

`InvocationKind.EXACTLY_ONCE` が 1 度だけ呼ばれることが保証されることで、`run` の `block` 内で変数を初期化が可能になります。

```kotlin
val x: Int
run {
    x = 0
}

println(x)
```

## `InvocationKind.EXACTLY_ONCE` の誤解

例えば、次のような `runCatching` 関数を考えます。この関数は `block` を実行し、その結果を `Result` 型で返します。`block` が例外をスローした場合は、`Result.failure` を返します。`runCatching` は `block` が 1 回呼ばれるため、`contract` に `InvocationKind.EXACTLY_ONCE` を指定してよいように感じます。

しかし、`InvocationKind.EXACTLY_ONCE` は「1 回呼ばれる」ことを意味するわけではありません。正確には「正常に 1 回完了する」ことを意味します。 つまり、`block` が 1 回呼ばれたとしても、その呼び出しが例外をスローして正常に完了しなかった場合は、`InvocationKind.EXACTLY_ONCE` の条件を満たしません。

```kotlin
inline fun <R> runCatching(block: () -> R): Result<R> {
    // 追加
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }

    return try {
        Result.success(block())
    } catch (e: Throwable) {
        Result.failure(e)
    }
}
```

もし `InvocationKind.EXACTLY_ONCE` を指定してしまうと、次のコードは問題なくコンパイルされます。しかし、`x` は初期化されないため、`pintln(x)` の時点では不定値になり予期せぬ動作を引き起こす可能性があります。

```kotlin
val x: Int
runCatching {
    throwableFunction() // ここで Exception が throw される
    x = 0
}

println(x)
```

幸いなことに現在の IDE[^ide] は賢いため `contract` に `InvocationKind.EXACTLY_ONCE` 設定した場合、条件を満たしていないことを検出し、“Wrong invocation kind 'EXACTLY_ONCE' for 'block: () -> R' specified, the actual invocation kind is 'AT_MOST_ONCE'.” と警告します。

[^ide]: Android Studio Otter 2 Feature Drop | 2025.2.2 Patch 1 の “Enable K2 mode” を有効にして確認しました。

## おわりに

TODO: ここに何か書く

## 参考文献

1. [kotlin.contracts | Core API – Kotlin Programming Language](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.contracts/)
2. [InvocationKind | Core API – Kotlin Programming Language](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.contracts/-invocation-kind/)
3. [KEEP/proposals/KEEP-0139-kotlin-contracts.md at main · Kotlin/KEEP](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0139-kotlin-contracts.md)
4. [Fixing Indeterminate Behavior Caused by `InvocationKind.EXACTLY_ONCE` by Daiji256 · Pull Request #106 · michaelbull/kotlin-result](https://github.com/michaelbull/kotlin-result/pull/106)
