---
title: "mapState, combineState: StateFlow を StateFlow へ変換したい"
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

# mapState, combineState: StateFlow を StateFlow へ変換したい

`StateFlow<Foo>` を `StateFlow<Bar>` に変換したい、`StateFlow<Foo>` と `StateFlow<Bar>` を `StateFlow<Pair<Foo, Bar>>` に変換したい、そういうことがあります。この記事ではそれを解決する 1 つの手段を紹介します。

## `Flow` と `StateFlow` の違いについて

cold と hot の違いについてかるーく説明する

```kotlin
public interface Flow<out T>
```

```kotlin
public interface StateFlow<out T> : SharedFlow<T>
public interface SharedFlow<out T> : Flow<T>
```

## `Flow` の `map` と `combine` について

`Flow` の `map` や `combine` は cold な `Flow` を前提に用意されている。普通に使うと `StateFlow` を `Flow` に変換することになる。

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

public fun <T1, T2, R> combine(
    flow: Flow<T1>,
    flow2: Flow<T2>,
    transform: suspend (T1, T2) -> R,
): Flow<R>
```

## `StateFlow` から `StateFlow` に変換したい

### `stateIn`

`Flow` を `StateFlow` に変換するための `stateIn` がある。しかしこれには `CoroutineScope` を使って、`collect` することで `StateFlow` に変換している。非同期処理が不要であっても、そうなってしまう。

```kotlin
public fun <T> Flow<T>.stateIn(
    scope: CoroutineScope,
    started: SharingStarted,
    initialValue: T
): StateFlow<T>
```

```kotlin
val foo: StateFlow<Foo> = TODO()
val bar: StateFlow<Bar> = foo
    .map { it.toBar() }
    .stateInt(
        scope = coroutineScope,
        started = SharingStarted.Lazily,
        initialValue = initialBar,
    )
```

### 同期処理で実現したい

こういう関数があれば良い。

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

fun <T1, T2, R> combineState(
    flow: StateFlow<T1>,
    flow2: StateFlow<T2>,
    transform: (T1, T2) -> R,
): StateFlow<R>
```

### 実装

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

## まとめ

## 参考文献

https://kotlinlang.org/docs/flow.html

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/
