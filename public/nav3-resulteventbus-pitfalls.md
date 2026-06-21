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

# （仮）Nav3のResultEventBusの微妙なポイント（alpha）

navigation3の1.2.0-alpha02で追加されたResultEventBusを軽く紹介

（記事中では1.2.0-alpha04で確認している）

コンセプトは微妙なポイント（注意点）を書くことだが、ひとまず主な機能・仕組み・使い方を簡単に説明する

概要を説明 → 微妙なポイント → まとめの順で説明する予定

ResultEventBusを使うことで、遷移先等から結果を受け取ることができる

```kotlin
val entries = rememberDecoratedNavEntries(
    entryDecorators = listOf(
        rememberResultEventBusNavEntryDecorator(),
    ),
    // ...
)
```

```kotlin
@Serializable
data class FooResult(
    val value: String?,
)
```

```kotlin
ResultEffect<FooResult> { result ->
    // ...
}
```

```kotlin
val resultEventBus = LocalResultEventBus.current
resultEventBus.sendResult(result = FooResult(value = "result"))
```

ChannelをMapで管理するという単純な仕組み

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

画面遷移などで自動でpopされることはない

```kotlin
public class ResultEventBusNavEntryDecorator<T : Any>(
    // ...
) :
    NavEntryDecorator<T>(
        onPop = {},
        // ...
    )
```

不要になったら自分でremoveしなければならない、そうしないとメモリリークする

A→B→C... のように遷移してA画面でresultを受け取るとしても、C→Bに結果を流す必要がなく、Cでsendすれば十分なのはラク  
また、X画面でresultをsendし、その後にY画面でresultを受け取るなども可能

Channelで保持しているので、受け取り（`ResultEffect` など）が複数ある場合、Fan-out形式で受け取ることになるため、必ずしも最新の結果を受け取るとは限らない

`ResultEventBusNavEntryDecorator` の実装から分かる通り、特に難しいことなく広いスコープで保持しているだけ

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

- https://developer.android.com/jetpack/androidx/releases/navigation3
- https://developer.android.com/reference/kotlin/androidx/navigation3/runtime/result/ResultEventBus
- https://mvnrepository.com/artifact/androidx.navigation3/navigation3-runtime
- https://kotlinlang.org/docs/channels.html
- https://github.com/androidx/androidx/tree/723b5af31e4c5dff62be1fe8684c73d73399602b/navigation3/navigation3-runtime/src/commonMain/kotlin/androidx/navigation3/runtime/result
