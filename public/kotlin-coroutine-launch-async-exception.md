---
title: launchとasyncの例外処理のタイミングの違いについて
tags:
  - Kotlin
  - 非同期処理
  - 例外処理
  - coroutines
private: false
updated_at: '2026-07-13T00:05:10+09:00'
id: 05681b3bc492898a5598
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

Kotlinのcoroutineを使って非同期処理を実装する際、coroutineを起動するために `launch` や `async` を使います。

非同期処理を開始するという点では同じです。`Job` を扱いたい場合は `launch`、`Deferred` により結果を受け取りたい場合は `async` のように使い分けるのが一般的です。

しかし、これらは例外（Exception）の扱いが異なります。

本記事では、`launch` や `async` の例外に対する振る舞いの違い、そして例外処理を行う適切なタイミングについて解説します。

## `launch` は即座に例外を伝播する

`launch` はブロック内で例外が発生したとき、積極的に例外をthrowします。この例外はcatchされなかった場合、即座に親coroutineへ伝播します。

```kotlin
scope.launch {
    // 例外が発生した瞬間、即座に親に伝播する
    throwableFunction() // 例: throw Exception("例外")
}
```

## `async` は `await()` が呼ばれるまで例外を保持する

`async` ブロック内で例外が発生した場合、その場では例外をthrowしません。代わりに、戻り値である `Deferred` 内に例外が発生したことを保持します。

ただし、この挙動が成立するのは `supervisorScope` などを用いて親への例外伝播を制御している場合、もしくはトップのcoroutineである場合のみです。通常のスコープで子coroutineとして起動された場合、coroutineの仕様（構造化並行性）により `await()` を待たずに即座に親へ例外が伝播します。

例外の伝播を適切に防いだ状態であれば、`Deferred` から結果を取り出すために `await()` を呼び出したタイミングで、初めて例外がthrowされます。

```kotlin
val deferred = scope.async {
    // ここで例外が発生しても、即座に伝播せずDeferredに保持される
    throwableFunction()
}

// 結果を取り出そうとしたときに、例外がthrowされる
deferred.await()
```

## `produce` は `receive()` が呼ばれるまで例外を保持する

実は、`launch` と `async` の他に `produce` もあります[^ExperimentalCoroutinesApi]。

[^ExperimentalCoroutinesApi]: `produce` は `ExperimentalCoroutinesApi` のため、採用されるシーンは `launch` と `async` に比べて少ないでしょう。

`produce` の戻り値は `ReceiveChannel` で、例外については `async` の `Deferred` に似た特徴を持ちます。例外を `ReceiveChannel` に保持し、`receive()` を呼び出した時に例外がthrowされます。

```kotlin
val receiveChannel = scope.produce {
    // ここで例外が発生しても、即座に伝播せずReceiveChannelに保持される
    send(throwableFunction()) 
}

// 要素を受け取ろうとしたときに、例外がthrowされる
receiveChannel.receive()
```

## `try`/`catch` や `runCatching` はどこに書くべきか

この挙動の違いを最も実感するのが、例外をキャッチするための `try`/`catch` や `runCatching` の位置です。

### `launch` の場合

`launch` は即座に例外をthrowするため、ラムダ内にある処理そのものに対して例外処理が必要です。

```kotlin
scope.launch {
    try {
        throwableFunction()
    } catch (e: Exception) {
        // 例外のハンドリング
    }
}
```

### `async` の場合

`async` の場合、必ずしもラムダ内で例外処理をする必要はなく、例外がthrowされる `await()` の呼び出し箇所での例外処理が可能です。

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

`produce` の場合も `async` と同様に、値を受け取るタイミングで例外がthrowされるため、`receive()` の呼び出し箇所での例外処理が可能です。

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

- `launch` は例外が発生したときに即座に親に伝播する
- `async` や `produce` では例外は戻り値（`Deferred` / `ReceiveChannel`）に保持され、`await()` や `receive()` 実行時にthrowされる
- ただし、`async` や `produce` の例外処理を考える場合、構造化並行性などを意識する必要がある

## 参考文献

- [Coroutine exceptions handling | Kotlin Documentation](https://kotlinlang.org/docs/exception-handling.html)
