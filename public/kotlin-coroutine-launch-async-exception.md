---
title: newArticle001
tags:
  - Kotlin
  - Coroutines
  - 例外処理
  - 非同期処理
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# launchとasyncの例外処理のタイミングの違いについて

## はじめに

Kotlinのcoroutineを使って非同期処理を実装する際、coroutineを起動するために `launch` や `async` を使います。

非同期処理を開始するという点では同じです。`Job` を扱いたい場合は `launch`、`Deferred` により結果を受け取りたい場合は `async` のように使い分けるのが一般的です。

しかし、これらは例外（Exception）の扱いが異なります。

本記事では、`launch` や `async` の例外に対する振る舞いの違いと、例外処理のタイミングについて解説します。

## `launch` は即座に例外を伝播する

`launch` はブロック内で例外が発生したとき、積極的に例外をthrowします。この例外はcatchされなかった場合に即座に親 `Job` に伝播します。

```kotlin
coroutineScope.launch {
    // 例外が発生したとき、即座に親Jobに伝搬する
    throwableFunction() // throw Exception("exception")
}
```

## `async` は `await()` が呼ばれるまで例外を保持する

`async` ブロック内で例外が発生した場合、その場では例外をthrowしません。代わりに、戻り値である `Deferred` に例外がthrowされたことを保持します。

そして、`Deferred` から結果を取り出すために `await()` を呼び出したタイミングで、初めて例外がthrowされます。

```kotlin
val deferred = coroutineScope.async {
    // ここで例外が発生しても、即時に伝搬しない
    throwableFunction() // throw Exception("exception")
}

// 結果を取り出そうとしたときに、例外がthrowされる
deferred.await()
```

## `produce` は `receive()` が呼ばれるまで例外を保持する

実は、`launch` と `async` の他に `produce` というものもあります。これは `ExperimentalCoroutinesApi` なので、`launch` と `async` に比べれば使われるシーンは少ないです。

`produce` の戻り値は `ReceiveChannel` で、例外については `Deferred` に似た特徴を持ちます。`ReceiveChannel` に例外を保持し、`receive()` を呼び出した時に例外がthrowされます。

```kotlin
val receiveChannel = coroutineScope.produce {
    // ここで例外が発生しても、即時に伝搬しない
    send(throwableFunction()) // throw Exception("exception")
}

// 結果を取り出そうとしたときに、例外がthrowされる
receiveChannel.receive()
```

## `try`/`catch` や `runCatching` はどこに書くべきか

この挙動の違いを最も実感するのが、例外のキャッチ処理のための `try`/`catch` や `runCatching` の位置です。

### `launch` の場合

`launch` は即座に例外をthrowするため、ラムダ内にある処理そのものに対して例外処理が必要です。

```kotlin
coroutineScope.launch {
    try {
        throwableFunction()
    } catch (e: Exception) {
        // 例外のハンドリング
    }
}
```

### `async` の場合

`async` の場合、必ずしもラムダ内にある処理で例外処理をする必要はなく、例外が放出される `await()` の呼び出し箇所で例外処理が可能です。

```kotlin
coroutineScope.launch {
    try {
        deferred.await()
    } catch (e: Exception) {
        // 例外のハンドリング
    }
}
```

### `produce` の場合

`produce` の場合、`async` と同様に値を受け取るタイミングで例外処理が必要なため、`receive()` の呼び出します。

```kotlin
coroutineScope.launch {
    try {
        receiveChannel.receive()
    } catch (e: Exception) {
        // 例外のハンドリング
    }
}
```

## まとめ

- `launch` は例外が発生したときに親 `Job` に伝搬するのに対して、`async` では伝搬されない
- `async` では例外は戻り値の `Deferred` に保存され、`await()` 実行時にthrowされる
- `produce` では例外は戻り値の `ReceiveChannel` に保存され、`receive()` 実行時にthrowされる

## 参考文献

- [Coroutine exceptions handling | Kotlin Documentation](https://kotlinlang.org/docs/exception-handling.html)
