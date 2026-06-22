---
title: Nav3のResultEventBusの機能と微妙なポイント（alpha版）
tags:
  - Android
  - Kotlin
  - Compose
  - Navigation3
  - ResultEventBus
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

:::note warn
本記事の内容はalpha版のソースコードで確認しています。今後のアップデートで大きく変わる場合があります。
:::

## はじめに

Jetpack Navigation 3（Nav3）の1.2.0-alpha02[^1.2.0-alpha04]にて、画面間の結果受け渡しをサポートする新しいAPIとして `ResultEventBus` が追加されました。

[^1.2.0-alpha04]: 本記事は執筆時点で最新の1.2.0-alpha04で確認しています。概ね1.2.0-alpha02からの変更はありません。

本記事では、この `ResultEventBus` の主な機能・仕組み・使い方を簡単に説明した上で、気になる微妙なポイントや注意点について解説します。

## `ResultEventBus` の基本的な使い方

### `NavEntryDecorator` のセットアップ

`NavDisplay` などのセットアップ時の `entryDecorators` に `rememberResultEventBusNavEntryDecorator()` を追加します。

```kotlin
entryDecorators = listOf(
    // ...
    rememberResultEventBusNavEntryDecorator(),
)
```

### 結果として送受信するデータの型の定義

結果として送受信するデータの型を定義します。

```kotlin
data class FooResult(
    val value: String?,
)
```

### 受信側の実装

結果の受け取りには `ResultEffect` もしくは `conflateAsState` を使用します。`ResultEffect` では、結果をラムダで受け取ることができます。`conflateAsState` は最新の結果を `State` に格納します。

一度きりのイベント（スナックバーの表示など）として消費する場合は `ResultEffect` を、UIの状態として保持し続けたい場合は `conflateAsState` を利用すると良いでしょう。

```kotlin
ResultEffect<FooResult> { result ->
    // ...
}
```

```kotlin
val result by LocalResultEventBus.current.conflateAsState(
    initialValue = // ...
)
```

### 送信側の実装

`ResultEventBus` の `sendResult()` で結果を送信します。`ResultEventBus` のインスタンスは `LocalResultEventBus` で取得できます。

```kotlin
val resultEventBus = LocalResultEventBus.current
resultEventBus.sendResult(result = FooResult(value = "result"))
```

## `ResultEventBus` の仕組み

`ResultEventBus` の実装は非常にシンプルです。キーに紐づくように複数の `Channel` を保持しているだけです。

送信されたデータはこの `Channel` に保持され、`ResultEffect` や `conflateAsState` は `getResultFlow` を経由して結果を収集します。

```kotlin
public class ResultEventBus {
    internal val channelMap: SnapshotStateMap<String, Channel<Any?>> = mutableStateMapOf()

    internal fun getResultFlow(resultKey: String): Flow<Any?> {
        if (!channelMap.contains(resultKey)) {
            channelMap[resultKey] =
                Channel(capacity = BUFFERED, onBufferOverflow = BufferOverflow.SUSPEND)
        }
        return channelMap[resultKey]!!.receiveAsFlow()
    }

    public fun removeResult(resultKey: String) {
        channelMap.remove(resultKey)?.close()
    }

    // ...
}
```

## 微妙なポイント（注意点）

### 自動でクリーンアップされない（メモリリークの懸念）

`ResultEventBusNavEntryDecorator` の実装から分かる通り、全ての `entry` で同一の `ResultEventBus` を参照しているだけです。ナビゲーションのポップに連動することもありません。

もう受け取ることがないとしても、明示的に `removeResult()` を呼び出さない限り、送受信するための `Channel` は残り続けてしまいます。

```kotlin
public class ResultEventBusNavEntryDecorator<T : Any>(
    private val bus: ResultEventBus = ResultEventBus()
) :
    NavEntryDecorator<T>(
        onPop = {},
        decorate = { entry ->
            CompositionLocalProvider(LocalResultEventBus provides bus) { entry.Content() }
        },
    )
```

### Activity再生成時のデータ消失に対応しない

自動で破棄されない性質により、A → B → Cのように深く遷移したとしても、Cで送信した結果を、Bを経由せずにAで受け取ることができます。また、Aで送信した結果をBで受け取ることもできます。

しかし、結果の送信と受信の間に間隔が開くことで、問題もあります。例えば、

1. Cで結果を送信し、Bに戻る
2. Bに滞在している間に、Activityが再生成される
3. Aに戻っても結果を受け取れない

ということがあります。これは、`ResultEventBus` がActivity再生成時のデータ消失に対応していないために起こることです。

また、イベントの送受信のタイミングが自由であることから、開発者自身がそれらのタイミングを正しく管理しなければなりません。

### Fan-out形式で分配

同じ型の結果を監視する `ResultEffect` や `conflateAsState` が複数同時に存在する場合、`Channel` の仕様によりFan-out形式でイベントが分配されます。

つまり、送信された1つの結果をすべてのオブザーバーが受け取れるわけではなく、どれか1つにしか到達しません。必ずしも最新の結果を受信できるとは限らない点に注意が必要です。

## まとめ

- Nav3の `ResultEventBus` は結果の受け渡しを簡潔かつ型安全に実現する便利な仕組み
- `Channel` による保持のため、Activity再生成時のデータ消失や不要な結果の削除などのメモリ管理やライフサイクルには注意が必要
- イベントのライフサイクルや多重受信がないようにするなど、実装時に注意する必要がある

## 参考文献

- [navigation3 | Jetpack | Android Developers](https://developer.android.com/jetpack/androidx/releases/navigation3)
- [ResultEventBus | API reference | Android Developer](https://developer.android.com/reference/kotlin/androidx/navigation3/runtime/result/ResultEventBus)
- [androidx/navigation3/runtime/result | androidx/androidx | GitHub](https://github.com/androidx/androidx/tree/723b5af31e4c5dff62be1fe8684c73d73399602b/navigation3/navigation3-runtime/src/commonMain/kotlin/androidx/navigation3/runtime/result)
- [Channels | Kotlin Documentation](https://kotlinlang.org/docs/channels.html)
