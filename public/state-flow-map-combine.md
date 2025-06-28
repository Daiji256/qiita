---
title: "mapState / combineState: StateFlow を同期処理で変換・合成する"
tags:
  - Android
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

# mapState / combineState: StateFlow を同期処理で変換・合成する

Android アプリ開発において、`StateFlow` は UI 状態管理に不可欠です。しかし、実際の開発では以下の課題に直面することがあります：

- `StateFlow` から別の `StateFlow` への変換
- 複数の `StateFlow` を組み合わせた `StateFlow` の作成

本記事では、これらの課題を解決する **`mapState`** と **`combineState`** を提案し、実際の Android アプリ開発での活用例を紹介します。

## 標準的なアプローチの問題点

`StateFlow` を別の `StateFlow` に変換する標準的なアプローチ（従来手法）では、`map` / `combine` と `stateIn` を組み合わせて使用します：

```kotlin
class UserViewModel : ViewModel() {
    private val user = MutableStateFlow(User(name = "太郎", age = 25))

    val userName: StateFlow<String> = user
        .map { it.name } // Flow<String> になる
        .stateIn(
            scope = viewModelScope, // CoroutineScope が必要
            started = SharingStarted.WhileSubscribed(5000), // 用途ごとに選択
            initialValue = user.value.name, // 初期値の重複定義
        )
}
```

従来手法にはいくつかの問題があります：

- **`CoroutineScope` への依存**：コルーチンを起動する必要がある
- **メモリ使用量の増加**：変換後の値をメモリに保持するため、メモリ使用量が増える
- **コルーチンによるオーバーヘッド**：非同期処理に伴うオーバーヘッドが発生する
- **実装量が多い**：`started` の選択や初期値の定義など、実装量が多い

## 同期変換関数の実装

提案する解決策は以下の要件を満たします：

- **軽量性**：`CoroutineScope` や追加リソースが不要
- **効率性**：必要時のみ計算を実行し、メモリにキャッシュしない
- **互換性**：`StateFlow` との完全な互換性がある

`mapState` / `combineState` は `StateFlow` を直接実装し、値を参照する際に `transform` を実行するように実装されます：

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

## Android アプリ開発での実用例

### `mapState` による変換例

従来手法の `map` を `mapState` に、`stateIn` を削除することで、提案手法に移行できます：

```kotlin
class UserProfileViewModel : ViewModel() {
    private val user = MutableStateFlow(User(name = "太郎", age = 25))

    // 従来手法
    val userNameLegacy = user
        .map { it.name }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = user.value.name,
        )

    // 提案手法
    val userName = user
        .mapState { it.name }
}
```

### `combineState` による合成例

`mapState` と同様に、従来手法の `combine` を `combineState` に、`stateIn` を削除することで、提案手法に移行できます：

```kotlin
class UserEditViewModel : ViewModel() {
    private val name = MutableStateFlow("")
    private val age = MutableStateFlow(0)

    // 従来手法
    val userLegacy = combine(name, age) { name, age ->
        User(name = name, age = age)
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = User(name = name.value, age = age.value),
    )

    // 提案手法
    val user = combineState(name, age) { name, age ->
        User(name = name, age = age)
    }
}
```

## パフォーマンス分析と使い分けガイド

### 処理性能

`map` / `combine` + `stateIn`（従来手法）と `mapState` / `combineState`（提案手法）の処理性能を比較すると、次の通りです：

| 項目             | `map` / `combine` + `stateIn`    | `mapState` / `combineState` |
| :--------------- | :------------------------------- | :-------------------------- |
| CPU 使用量       | **低（キャッシュ済み値を返却）** | 高（参照時に計算）          |
| メモリ使用量     | 高（変換結果をキャッシュ）       | **低（遅延評価）**          |
| 初期化コスト     | 高（コルーチン起動）             | **低（オブジェクト作成）**  |
| `CoroutineScope` | 依存する                         | **依存しない**              |

### 適切な使い分け

提案手法は参照するたびに変換処理を実行します。そのため、変換処理は**純粋関数（副作用なし）**で**軽量**である必要があります。また、高頻度の `value` へのアクセスは避けるべきです。

提案手法の `mapState` / `combineState` は次のような場合に有用です：

```kotlin
val displayName = user.mapState { "${it.firstName} ${it.lastName}" }
val isActive = combineState(now, endAt) { now < endAt }
```

一方で、重い処理や副作用を伴う場合には、従来手法の `map` / `combine` + `stateIn` が適しています：

```kotlin
val sortedUsers = users
    .map { it.sortedBy { it.name } } // 重い処理
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = null,
    )
```

## まとめ

`StateFlow` を変換・合成するために `map` / `combine` + `stateIn` を利用する場合の問題を指摘しました。そして、その問題を解決するための `mapState` と `combineState` を提案しました。

提案手法では多くの場面で、少ない実装量で高パフォーマンスを実現できます。

## 参考文献

https://kotlinlang.org/docs/flow.html

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/

https://github.com/Daiji256/android-showcase/tree/e84f98bcec79c8c5aecae63cc4a97c7411d48d2c/core/common/src/main/kotlin/io/github/daiji256/showcase/core/common/stateflow
