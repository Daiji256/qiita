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

Android アプリ開発において、`StateFlow` は UI 状態管理に不可欠です。しかし、実際の開発では以下の課題に直面することがあります：

- `StateFlow` から別の `StateFlow` への変換
- 複数の `StateFlow` を組み合わせた `StateFlow` の作成

本記事では、これらの課題を解決する **`mapState`** と **`combineState`** を提案し、Android アプリ開発での活用例を紹介します。

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
            initialValue = user.value.name, // 初期値を重複して定義
        )
}
```

この従来手法には次のような問題があります：

- **`CoroutineScope` への依存**：コルーチンの起動が必要
- **メモリ使用量の増加**：変換後の値をメモリに保持するため
- **オーバーヘッド**：非同期処理に伴うコルーチン起動コストが発生する
- **実装量が多い**：`started` の指定や初期値の定義などが必要

## 同期変換関数の実装

今回提案する `mapState` / `combineState` は次の要件を満たします：

- **軽量性**：`CoroutineScope` や追加リソースが不要
- **効率性**：必要なときに変換処理を実行し、メモリにキャッシュしない
- **互換性**：`StateFlow` との完全な互換性

`mapState` / `combineState` は `StateFlow` を直接実装し、値を参照する際に `transform` を実行するように動作します[^suppress-optin]：

[^suppress-optin]: `StateFlow` の実装は不安定・実験的なため、`@Suppress("UnnecessaryOptInAnnotation")` と `@OptIn(ExperimentalForInheritanceCoroutinesApi::class)` が必要です：[kotlinx.coroutines Issue #3770](https://github.com/Kotlin/kotlinx.coroutines/issues/3770)

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

## Android アプリ開発での利用例

### `mapState` による変換例

従来手法で `map` + `stateIn` を使っていた部分を `mapState` に置き換え、実装を簡素化できます：

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

複数の `StateFlow` を組み合わせる場合も同様です：

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

## 処理内容

map / combine + stateIn（従来手法）と mapState / combineState（提案手法）の処理内容をまとめると次の通りです：

| 項目             | `map` / `combine` + `stateIn` | `mapState` / `combineState` |
| :--------------- | :---------------------------- | :-------------------------- |
| CPU 使用量       | 低（キャッシュ済み値）        | 高（参照時に計算）          |
| メモリ使用量     | 高（キャッシュ保持）          | 低（都度計算）              |
| 初期化コスト     | 高（コルーチン起動）          | 低（オブジェクト作成）      |
| `CoroutineScope` | 必要                          | 不要                        |

## 従来手法と提案手法の使い分け

従来手法と提案手法はそれぞれにメリットがあります。次の条件を満たすときに提案手法を選択することを推奨します：

- 変換処理が軽い場合（文字列結合、数値計算など）
- 同期処理で実行したい
- `value` の参照頻度が低い

以下の場合は従来手法を利用してください：

- 変換処理が重い場合（リストのソートや画像処理など）
- 副作用を含む（ログ出力、データベース更新など）

## まとめ

本記事では、`StateFlow` の変換・合成における従来手法の課題を整理し、それを解決する `mapState` / `combineState` を提案しました。提案手法は、実装量の削減、メモリ効率の向上、初期化コストの低減を実現しつつ、適切な条件下で効率的に動作します。変換処理の特性やアクセス頻度に応じて使い分けることで、より柔軟で効率的な UI 状態管理が可能になります。

## 参考文献

https://kotlinlang.org/docs/flow.html

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/

https://github.com/Kotlin/kotlinx.coroutines/issues/3770

https://github.com/Daiji256/android-showcase/tree/e84f98bcec79c8c5aecae63cc4a97c7411d48d2c/core/common/src/main/kotlin/io/github/daiji256/showcase/core/common/stateflow
