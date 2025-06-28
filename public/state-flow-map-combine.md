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

Kotlin Coroutines の `StateFlow` は、UI 状態管理などでよく使われます。しかし、例えば以下のような状況で少し苦労します。

- `StateFlow<User>` から `StateFlow<UserId>` のように、特定のプロパティだけを取り出したい
- 複数の `StateFlow` を合成して、1 つの `StateFlow<UiState>` にまとめたい

これを実現しようとすると、通常は `CoroutineScope` などを使った冗長な処理が必要です。この記事では、`StateFlow` から新しい `StateFlow` を**同期的に**変換・合成する関数 `mapState` と `combineState` を紹介します。

## `Flow` と `StateFlow` の違い

なぜこうした問題や解決策が必要かを理解するために、まず `Flow` と `StateFlow` の違いを確認します。これにより、従来の変換アプローチの問題点が明確になります。

- `Flow` は Cold Stream であり、`collect` されるまで処理が始まらない
- `StateFlow` は Hot Stream で、常に 1 つの `value` を保持しており、`CoroutineScope` なしに即座に値を取得できる

## 一般的な変換方法とその課題

`map` や `combine` と `stateIn` を使うことで、`StateFlow` の変換は可能ですが、いくつか問題があります。

- `map` や `combine` は `Flow` を返すだけで、即座に `value` を取得できない
- `stateIn` で `StateFlow` に変換する場合、必ず `CoroutineScope` が必要となる

こうした処理は以下のように記述されます。

```kotlin
val bar: StateFlow<Bar> = foo
    .map { it.toBar() }
    .stateIn(
        scope = coroutineScope,
        started = SharingStarted.Lazily,
        initialValue = initialBar,
    )
```

## 同期処理で変換・合成する関数

以下のような関数を実装することで、`CoroutineScope` を使わず同期的に変換・合成できます。`StateFlow` を直接実装することで、`CoroutineScope` を使わずに `StateFlow` を作成できます。

`value` や `collect` で値を参照するときに `transform` により変換を実行します。これにより常に最新の変換後の値を同期的に得ることができます。また、`distinctUntilChanged()` により、`StateFlow` の連続して同じ値を流さないことを満たすことができます。

ただし、この実装では `value` を参照するたびに `transform` が呼ばれます。そのため、**transform は軽量で副作用のない関数にすべき**です。重い処理が必要な場合は、従来の `map` や `combine` と `stateIn` を用いたアプローチを選択してください。

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

今回紹介した `mapState` と `combineState` により以下が実現できます。ただし、変換処理のコストや副作用には注意してください。

- シンプルで直感的な記述での変換・合成
- `CoroutineScope` 不要
- `value` プロパティで即座に最新の状態を取得可能

## 参考文献

https://kotlinlang.org/docs/flow.html

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/
