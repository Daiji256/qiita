---
title: "mapState, combineState: StateFlow を同期処理で変換・合成する"
tags:
  - Kotlin
  - Flow
  - StateFlow
  - Coroutine
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# mapState, combineState: StateFlow を同期処理で変換・合成する

Kotlin Coroutines の `StateFlow` は、UI の状態管理などで非常に役に立ちます。しかし、`StateFlow<User>` から `StateFlow<UserId>` のように `User` から `UserId` だけを取り出す変換をしたい、`StateFlow<FooState>` と `StateFlow<BarState>` などを合成して `StateFlow<UiState>` を作りたい、といった場面で `CoroutineScope` を使うなど冗長な処理が必要になります。

この記事では、`StateFlow` から新たな `StateFlow` を `CoroutineScope` を使わず、同期的に変換・合成するための拡張関数 `mapState` と `combineState` を実装し紹介します。

## `Flow` と `StateFlow` の違い

これらの関数の必要性を理解するために、まず `Flow` と `StateFlow` の基本的な性質の違いをおさらいします。

`Flow` は Cold Stream で `collect` されるまで処理が実行されません。

`StateFlow` は Hot Stream で `Flow` を継承しています。`StateFlow` は常に 1 つの `value` を保持します。`value` は `CoroutineScope` がなしに即時に参照可能です。

```kotlin
public interface Flow<out T>
```

```kotlin
public interface StateFlow<out T> : SharedFlow<T>
public interface SharedFlow<out T> : Flow<T>
```

## 一般的な変換方法とその問題

一般的に `StateFlow` を別の `StateFlow` に変換する場合、`map` や `combine` と `stateIn` を用います。これにはいくつかの問題があります。

### `map` や `combine` は `Flow` を返す

`Flow` には `map` や `combine` といった変換や合成を行うための関数が用意されています。しかし、これらは `Flow` を返すように設計されているため、`StateFlow` に対しても利用したとしても `Flow` になってしまいます。

`StateFlow` が持つ `value` に `CoroutineScope` なしに即時に参照できるという特性が失われてしまいます。

```kotlin
public inline fun <T, R> Flow<T>.map(
    crossinline transform: suspend (value: T) -> R,
): Flow<R>
```

```kotlin
public inline fun <reified T, R> combine(
    vararg flows: Flow<T>,
    crossinline transform: suspend (Array<T>) -> R,
): Flow<R>
```

### `stateIn` による `StateFlow` への再変換

`Flow` を `StateFlow` に変換するための `stateIn` があります。

```kotlin
public fun <T> Flow<T>.stateIn(
    scope: CoroutineScope,
    started: SharingStarted,
    initialValue: T,
): StateFlow<T>
```

`stateIn` により、`StateFlow` への変換は実現できますが、`Flow` を `collect` するために `CoroutineScope` を必要とします。つまり、変換が同期処理であったとしても、コルーチンを起動し変換することになります。

```kotlin
val foo: StateFlow<Foo> = TODO()
val bar: StateFlow<Bar> = foo
    .map { it.toBar() }
    .stateIn(
        scope = coroutineScope,
        started = SharingStarted.Lazily,
        initialValue = initialBar,
    )
```

## 同期処理で変換・合成する

そこで、以下のような同期処理により `StateFlow` のまま変換・合成する関数がほしいです。

```kotlin
inline fun <T, R> StateFlow<T>.mapState(
    crossinline transform: (T) -> R,
): StateFlow<R>
```

```kotlin
inline fun <reified T, R> combineState(
    vararg flows: StateFlow<T>,
    crossinline transform: (Array<T>) -> R,
): StateFlow<R>
```

この関数があればシンプルに変換・合成処理を実装できます。

```kotlin
val foo: StateFlow<Foo> = TODO()
val bar: StateFlow<Bar> = foo.mapState { it.toBar() }
```

### 実装

この理想を実現する実装がこちらです。`StateFlow` インターフェースを直接実装することで、`CoroutineScope` を使わずに `StateFlow` を作成できます。

`value` や `collect` で値を参照するときに `transform` により変換を実行します。これにより常に最新の変換後の値を同期的に得ることができます。また、`distinctUntilChanged()` により、`StateFlow` の連続して同じ値を流さないことを満たすことができます。

しかしこの実装は、`value` を参照するたびに `transform` が実行されます。`transform` を軽量な処理に保つ必要があり、副作用を持たせてはいけません。ほとんどのケースでは問題にならないはずです。

もし `transform` が軽量ではない場合は、これまで通り `map` や `combine` と `stateIn` を使う方が好ましいです。

```kotlin
inline fun <T, R> StateFlow<T>.mapState(
    crossinline transform: (T) -> R,
): StateFlow<R> {
    val source = this

    @Suppress("UnnecessaryOptInAnnotation")
    @OptIn(ExperimentalForInheritanceCoroutinesApi::class)
    return object : StateFlow<R> {
        override val value: R
            get() = transform(source.value)

        override val replayCache: List<R>
            get() = listOf(value)

        override suspend fun collect(collector: FlowCollector<R>): Nothing {
            source
                .map(transform = transform)
                .distinctUntilChanged()
                .collect(collector = collector)
            awaitCancellation()
        }
    }
}
```

```kotlin
inline fun <reified T, R> combineState(
    vararg flows: StateFlow<T>,
    crossinline transform: (Array<T>) -> R,
): StateFlow<R> {
    @Suppress("UnnecessaryOptInAnnotation")
    @OptIn(ExperimentalForInheritanceCoroutinesApi::class)
    return object : StateFlow<R> {
        override val value: R
            get() = transform(flows.map { it.value }.toTypedArray())

        override val replayCache: List<R>
            get() = listOf(value)

        override suspend fun collect(collector: FlowCollector<R>): Nothing {
            combine(flows = flows, transform = transform)
                .distinctUntilChanged()
                .collect(collector = collector)
            awaitCancellation()
        }
    }
}
```

2 つの `StateFlow` を対象にした `combineState` は以下のように実装できます。

```kotlin
fun <T1, T2, R> combineState(
    flow: StateFlow<T1>,
    flow2: StateFlow<T2>,
    transform: (T1, T2) -> R,
): StateFlow<R> =
    combineState(flow, flow2) { args: Array<*> ->
        @Suppress("UNCHECKED_CAST")
        transform(
            args[0] as T1,
            args[1] as T2,
        )
    }
```

## まとめ

`StateFlow` は強力ですが、その変換には時として冗長なコードが必要でした。今回紹介した `mapState` と `combineState` はそれらの問題を解決する以下のメリットを提供します。

- シンプルで直感的な実装で変換・合成が可能
- `CoroutineScope` を必要としない
- `value` プロパティを通じて同期的に最新の状態を取得できる

## 参考文献

https://kotlinlang.org/docs/flow.html

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/
