---
title: Navigation Compose における ViewModel での引数の受け取り方 4 選と比較
tags:
  - Android
  - Kotlin
  - Jetpack
  - Hilt
  - DI
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# Navigation Compose における ViewModel での引数の受け取り方 4 選と比較

## はじめに

Navigation Compose 2.8.0 以降では、シリアル化可能なクラスを用いて、型安全に画面間でデータを受け渡すことが可能になりました。従来の文字列ベースの `route` 指定とは異なり、引数の受け渡しに関する記述がより宣言的で安全になっています。

本記事では、「画面遷移時に渡された引数を ViewModel でどのように受け取るか」に焦点をあて、4 つの方法を紹介し比較します。すべて Hilt + Navigation Compose 2.8.0 以降の環境を想定していますが、他の DI フレームワークやバージョンでも応用可能です。

## 引数の渡し方

Navigation Compose では、引数を含むシリアル化可能なクラスを `composable` のルートとして利用できます：

```kotlin
@Serializable
data class SampleRoute(val arg: String)

fun NavGraphBuilder.sample() {
    composable<SampleRoute> { /* ... */ }
}
```

遷移元では、この `SampleRoute` を引数として `navigate()` を呼び出します：

```kotlin
fun NavController.navigateToSample(arg: String) =
    navigate(route = SampleRoute(arg = arg))
```

## 引数の受け取り方

遷移先では、`NavBackStackEntry` などを通じて `route` を取得します。その受け取り方と ViewModel での扱い方を紹介します。

### 1. Composable から ViewModel に渡す（関数呼び出し）

`NavGraphBuilder.composable()` の `content` ラムダで受け取る `NavBackStackEntry` から、`toRoute()` により `route` を取得します。そして ViewModel の関数を呼びだすことで `route` を ViewModel に渡します。

以下の例では `LifecycleEventEffect` を利用し、`ON_CREATE` イベントのタイミングで ViewModel に `route` を渡しています。

#### ✅ メリット

- 最小の知識で実装可能
- 引数を渡すタイミングを制御できるため、初期化を制御できる

#### ⚠️ デメリット

- `ViewModel` の初期化時にはまだ引数が使えない
- ライフサイクルが複雑な場合は制御が難しい

#### 実装例

```kotlin
fun NavGraphBuilder.sample() {
    composable<SampleRoute> { entry ->
        val route = entry.toRoute<SampleRoute>()
        SampleScreen(route = route)
    }
}
```

```kotlin
@Composable
fun SampleScreen(
    route: SampleRoute,
    viewModel: SampleViewModel = hiltViewModel(),
) {
    LifecycleEventEffect(event = Lifecycle.Event.ON_CREATE) {
        viewModel.onCreate(route = route)
    }
    // ...
}
```

```kotlin
@HiltViewModel
class SampleViewModel @Inject constructor() : ViewModel() {
    private val _arg = MutableStateFlow<String?>(null)
    val arg = _arg.asStateFlow()

    fun onCreate(route: SampleRoute) {
        _arg.value = route.arg
    }
}
```

### 2. Assisted Injection を使って ViewModel 生成時に渡す

ViewModel の初期化タイミングで引数を渡したい場合は、Dagger の Assisted Injection（Dagger による部分的な依存性注入を可能にする仕組み）が有効です。

Assisted Injection を利用することで、一部の依存関係を Dagger による依存解決に任せつつ、別の引数を直接渡すことができます。ViewModel の生成時に `route` を渡すことで、初期化時に `route` が利用できます。

#### ✅ メリット

- 引数を ViewModel 初期化に利用できる
- テスト時も依存関係を制御できる

#### ⚠️ デメリット

- `Factory` インターフェースや DI 設定が少し煩雑
- Assisted Injection の理解が必要

#### 実装例

```kotlin
@HiltViewModel(assistedFactory = SampleViewModel.Factory::class)
class SampleViewModel @AssistedInject constructor(
    @Assisted route: SampleRoute,
) : ViewModel() {
    val arg = route.arg

    @AssistedFactory
    interface Factory {
        fun create(route: SampleRoute): SampleViewModel
    }
}
```

```kotlin
@Composable
fun SampleScreen(
    route: SampleRoute,
    viewModel: SampleViewModel =
        hiltViewModel { factory: SampleViewModel.Factory ->
            factory.create(route = route)
        },
) {
    // ...
}
```

### 3. ViewModel で `SavedStateHandle` から受け取る

`SavedStateHandle` から `route` を抽出できます。これにより、Composable を介さずに ViewModel 側で直接引数を受け取ることができます。

`SavedStateHandle.toRoute()` は内部で `android.os.Bundle` を利用しているため、ユニットテストでは注意が必要です。`android.os.Bundle` は Android フレームワークのクラスであり、JVM 単体ではテストできません。JVM 上でシミュレートするために Robolectric が必要になります。また、`savedStateHandle["arg"] = "value"` のように引数を指定する必要があります。

#### ✅ メリット

- ViewModel の初期化と同時に取得できる
- 実装がシンプルでコード量が少ない

#### ⚠️ デメリット

- `android.os.Bundle` に依存しておりユニットテストでは Robolectric が必要
- テストでは `savedStateHandle["arg"] = "value"` のように準備する必要がある

#### 実装例

```kotlin
@HiltViewModel
class SampleViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
) : ViewModel() {
    private val route = savedStateHandle.toRoute<SampleRoute>()
    val arg = route.arg
}
```

```kotlin
@RunWith(RobolectricTestRunner::class)
class SampleViewModelTest {
    @Test
    fun test_dummy_arg() {
        val savedStateHandle = SavedStateHandle()
        savedStateHandle["arg"] = "dummy"
        val viewModel = SampleViewModel(
            savedStateHandle = savedStateHandle,
        )
        assertEquals(viewModel.arg, "dummy")
    }
}
```

### 4. `SavedStateHandle.toRoute()` を外で呼び、Hilt で注入

テストしやすさを重視する場合、`SavedStateHandle.toRoute()` を切り出して、ViewModel には `route` を直接注入する必要があります。

具体的には、`SavedStateHandle.toRoute()` を実行する Hilt Module を定義します。これにより、ViewModel は `route` を直接インジェクトできるため、ViewModel が `android.os.Bundle` に依存せず、ユニットテストをより簡単に記述できます。

#### ✅ メリット

- ViewModel が Android フレームワークに依存しないため、ユニットテストが容易
- DI による構成がシンプルで、再利用性が高い

#### ⚠️ デメリット

- Hilt Module の記述が必要
- Hilt に慣れていないとやや冗長に感じる

#### 実装例

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
object SampleRouteModule {
    @Provides
    fun providesSampleRoute(savedStateHandle: SavedStateHandle): SampleRoute =
        savedStateHandle.toRoute<SampleRoute>()
}
```

```kotlin
@HiltViewModel
class SampleViewModel @Inject constructor(
    route: SampleRoute,
) : ViewModel() {
    val arg = route.arg
}
```

## まとめ

以下は ViewModel での引数受け取りにおける 4 手法の比較表です。

| 方法               | 初期化タイミング | テストのしやすさ          | 実装の複雑さ          |
| ------------------ | ---------------- | ------------------------- | --------------------- |
| 関数呼び出し       | 遅延初期化       | ✅ 初期化を制御可能       | 🟡 関数呼び出しが必要 |
| Assisted Injection | 即時初期化       | ✅ 依存性注入でテスト容易 | 🟡 Factory が必要     |
| SavedStateHandle   | 即時初期化       | ⚠️ Bundle に依存          | 🟢 最小の実装量       |
| Hiltで provides    | 即時初期化       | ✅ 依存性注入でテスト容易 | 🟡 Module が必要      |

## おわりに

Navigation Compose 2.8.0 以降では、引数の受け渡し方法が柔軟かつ安全になりました。本記事で紹介した方法は一例であり、特定のベストプラクティスがあるわけではありません。プロジェクトや開発チームに合わせて選択すべきです。

今後リリースが予定されている Navigation 3 では画面遷移が大きく変更されます。Navigation 3 は Composable を主軸に置いています。本記事で紹介した中では、特に序盤の方法が今後の標準的なアプローチになる可能性があります。

## 参考文献

https://github.com/Daiji256/android-showcase/tree/5eec36161350c0bd5f9a82718abbcb389b48e25a/feature/navigationarguments/src/main/kotlin/io/github/daiji256/showcase/feature/navigationarguments

https://developer.android.com/guide/navigation/design/type-safety

https://developer.android.com/guide/navigation/navigation-3

https://dagger.dev/hilt/

https://dagger.dev/dev-guide/assisted-injection.html
