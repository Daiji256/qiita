---
title: "mapState, combineState: StateFlow を同期処理で変換・合成する"
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

# mapState, combineState: StateFlow を同期処理で変換・合成する

## はじめに

Android アプリ開発において、`StateFlow` は UI 状態管理において必要不可欠です。しかし、実際の開発では以下のシーンで課題に直面します：

- `StateFlow` から別の `StateFlow` への変換
- 複数の `StateFlow` を組み合わせた `StateFlow` を作成

本記事では、これらの課題を解決する `mapState` と `combineState` を提案し、実際の Android アプリ開発での活用例を紹介します。

## 標準的なアプローチの問題

`StateFlow` を別の `StateFlow` に変換する標準的なアプローチ（従来手法）は `map`/`combine` と `stateIn` を組み合わせます：

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

この従来のアプローチには以下の問題があります：

- `CoroutineScope` への依存
- 変換後の値をメモリに保持するため、メモリ使用量が増加
- コルーチンによる非同期処理のオーバーヘッド
- `started` の選択や初期値の重複定義などの多くの実装

## 同期変換関数の実装

### 設計思想と要件

提案する解決策は以下の要件を満たします：

- **軽量性**：`CoroutineScope` や追加リソースが不要
- **効率性**：必要時のみ計算を実行し、メモリにキャッシュしない
- **互換性**：`StateFlow` との完全な互換性

### 実装詳細

`mapState` と `combineState` は `StateFlow` を直接実装し、値を参照時に `transform` を実行するように実装されます：

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

従来手法の `map` を `mapState` に差し替え、`stateIn` を削除することで提案手法に乗り換えることができます：

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

`mapState` と同様に、従来手法の `combine` を `combineState` に差し替え、`stateIn` を削除することで提案手法に乗り換えることができます：

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

### 処理性能比較

`map`/`combine` + `stateIn`（従来手法）と `mapState`/`combineState`（提案手法）を処理性能を比較するとのこのようになります：

| 項目             | `map`/`combine` + `stateIn`      | `mapState`/`combineState`  |
| ---------------- | -------------------------------- | -------------------------- |
| CPU 使用量       | **低（キャッシュ済み値を返却）** | 中（参照時に計算）         |
| メモリ使用量     | 高（変換結果をキャッシュ）       | **低（遅延評価）**         |
| 初期化コスト     | 高（コルーチン起動）             | **低（オブジェクト作成）** |
| `CoroutineScope` | 依存する                         | **依存しない**             |

### 適切な使い分け

**`mapState`/`combineState` が適している場面：**

- プロパティアクセスや基本的な演算（文字列結合、四則演算など）
- `value` へのアクセス頻度が低い場合

```kotlin
// 適している例
val displayName = user.mapState { "${it.firstName} ${it.lastName}" }
val isAdult = user.mapState { it.age >= 18 }
val formattedPrice = price.mapState { "¥${it:,}" }
```

**従来手法が適している場面：**

- 重い計算処理（リストのソート、IO 処理など）
- 副作用を伴う処理（ログ出力、分析イベント送信など）
- `value` への頻繁なアクセス

```kotlin
// 従来手法
val sortedUsers = users
    .map { it.sortedBy { it.name } } // 重い処理
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = null,
    )
```

### 注意すべき実装上のポイント

**変換関数の制約：**

- **純粋関数**である必要があります（副作用なし）
- **軽量な処理**に留める

## まとめ

`mapState` と `combineState` は、Android 開発における `StateFlow` の活用を大幅に簡素化します。

**主なメリット:**

- `CoroutineScope` 不要によるコードの簡潔性とテスタビリティの向上
- 遅延評価によるメモリ効率の最適化
- 既存の `StateFlow` との完全な互換性
- Android開発でよくある軽量な変換処理に最適

**導入時の考慮点:**

- 変換処理は軽量かつ純粋関数に限定
- 頻繁な `value` アクセスがある場合は従来の `stateIn` を検討
- 重い処理や副作用がある場合は適用しない

## 参考文献

https://kotlinlang.org/docs/flow.html

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/

https://github.com/Daiji256/android-showcase/tree/e84f98bcec79c8c5aecae63cc4a97c7411d48d2c/core/common/src/main/kotlin/io/github/daiji256/showcase/core/common/stateflow
