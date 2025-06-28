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

## はじめに

Kotlin の `StateFlow` は状態管理などのシーンで利用されています。しかし、以下のような場面で開発者は課題に直面することがあります：

- `StateFlow<User>` から `StateFlow<String>`（ユーザー名のみ）への変換
- 複数の `StateFlow` を組み合わせて単一の `StateFlow<UiState>` を作成
- `CoroutineScope` を必要としない軽量な状態変換

本記事では、これらの課題を解決する `mapState` と `combineState` という 2 つの関数を紹介し、実際の使用例を通じてその有効性を示します。

## 問題の背景：既存アプローチの制約

### `Flow` と `StateFlow` の特性

まず、なぜこの問題が発生するのかを理解するために、`Flow` と `StateFlow` の違いを確認しましょう：

| 特性           | Flow               | StateFlow                |
| -------------- | ------------------ | ------------------------ |
| ストリーム型   | Cold Stream        | Hot Stream               |
| 実行タイミング | `collect` 時に開始 | 常に実行                 |
| 現在値の取得   | 不可               | `value` で即座に取得可能 |
| 用途           | 非同期データ処理   | 状態管理                 |

### 従来の変換手法の問題点

標準的なアプローチでは `map`/`combine` と `stateIn` を組み合わせますが、以下の制約があります：

```kotlin
val userNameState: StateFlow<String> = userState
    .map { it.name } // Flow<String> になる
    .stateIn(
        scope = viewModelScope, // CoroutineScope が必要
        started = SharingStarted.Lazily,
        initialValue = userState.name // 初期値の重複定義
    )
```

そのため、これらの問題点があります：

- `CoroutineScope` に依存する
- 初期値の重複定義など、冗長な実装になる
- 変換処理がコルーチンで実行される
- メモリ使用量の増加（変換後の値をメモリで保持）

## 解決策：同期変換関数の実装

### 設計思想

提案する解決策は以下の原則に基づいています：

- `value` で最新の変換結果を即座に取得
- `CoroutineScope` や追加リソース不要
- `StateFlow` の仕様（重複値の非配信）を維持
- 値アクセス時の偏見により、メモリ使用量を維持

### 実装詳細

以下が実装コードです：

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

## 実用例：ViewModel での活用

### 単一状態の変換

```kotlin
class UserProfileViewModel : ViewModel() {
    private val user = MutableStateFlow(User(name = "太郎", age = 25))

    // 従来の方法
    val userNameLegacy = user.map { it.name }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = user.name,
        )

    // 新しい方法
    val userName = user.mapState { it.name }
}
```

### 複数状態の合成

```kotlin
class UserEditorViewModel : ViewModel() {
    private val name = MutableStateFlow("")
    private val age = MutableStateFlow(0)

    // 従来の方法
    val userLegacy = combine(name, age, ::User)
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = User(name = name.value, age = age.value),
        )

    // 新しい方法
    val user = combineState(name, age, ::User)
}
```

## パフォーマンス考慮事項

### パフォーマンス比較

従来手法の `map`/`combine` + `stateIn` と提案手法の `mapState`/`combineState` には処理の違いによりパフォーマンス等に差があります：

| アプローチ                  | メモリ使用量 | CPU 使用量       | `CoroutineScope` |
| --------------------------- | ------------ | ---------------- | ---------------- |
| `map`/`combine` + `stateIn` | 高           | 低（キャッシュ） | 依存             |
| `mapState`/`combineState`   | 低           | 中（都度計算）   | なし             |

### 使用上の注意点

`mapState`/`combineState` の利用が適して例：

- 軽量な計算処理（プロパティ参照、基本的な演算など）
- 低頻度のアクセス

`map`/`combine` + `stateIn` の利用が適して例：

- 重い計算処理（リストのソート、IO 処理など）
- 副作用を伴う処理（データベース更新）
- 頻繁な `value` へのアクセス

## まとめ

`mapState`/`combineState` は以下のメリットを提供します：

- `CoroutineScope` 不要でコードが簡潔
- 即座の値アクセス（`value` プロパティ）
- 既存の `StateFlow` との互換性を維持
- メモリ効率的な実装

しかし、いくつかの注意点があります：

- 変換処理は軽量である必要
- 副作用はのない変換処理である必要
- 頻繁なアクセスに注意

## 参考文献

https://kotlinlang.org/docs/flow.html

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/

https://github.com/Daiji256/android-showcase/tree/e84f98bcec79c8c5aecae63cc4a97c7411d48d2c/core/common/src/main/kotlin/io/github/daiji256/showcase/core/common/stateflow
