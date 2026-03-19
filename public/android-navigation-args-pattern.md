---
title: Navigation ComposeにおけるViewModelでの引数の受け取り方4選と比較
tags:
  - Android
  - Kotlin
  - DI
  - JetpackCompose
  - Hilt
private: false
updated_at: '2025-06-15T08:55:46+09:00'
id: a5bbb5408fc1ffa69694
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Navigation Compose 2.8.0以降では、シリアル化可能なクラスを用いて、型安全に画面間でデータを渡せるようになりました。

本記事では、「画面遷移時の引数を `ViewModel` でどの受け取るか」に焦点を当て、4つの方法を紹介し比較します。すべてHilt + Navigation Compose 2.8.0以降の環境を想定していますが、他のDIフレームワークやバージョンでも応用可能です。

## 引数の渡し方

Navigation Composeでは、引数を含むシリアル化可能なクラスを `composable` のルートとして利用できます：

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

遷移先では、`NavBackStackEntry` などを通じて `route` を取得します。その受け取り方と `ViewModel` での扱い方を紹介します。

### 1. Composableから `ViewModel` に渡す（関数呼び出し）

`NavGraphBuilder.composable()` の `content` ラムダで受け取る `NavBackStackEntry` から、`toRoute()` により `route` を取得します。そして `ViewModel` の関数を呼び出すことで `route` を `ViewModel` に渡します。

以下の例では `LifecycleEventEffect` を利用し、`ON_CREATE` イベントのタイミングで `ViewModel` に `route` を渡しています。

#### ✅ メリット

- 最小の知識で実装可能
- 引数を渡すタイミングを制御でき、初期化も柔軟に行える

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

### 2. Assisted Injectionを使って `ViewModel` 初期化時に渡す

`ViewModel` の初期化タイミングで引数を渡したい場合は、DaggerのAssisted Injection（部分的なDIを可能にする仕組み）が有効です。

Assisted Injectionを利用することで、一部の依存関係をDaggerにより解決しつつ、別の引数を直接渡すことができます。これにより `ViewModel` の初期化時に `route` が利用できます。

#### ✅ メリット

- 引数を `ViewModel` 初期化に利用できる
- テスト時にDIできる

#### ⚠️ デメリット

- `Factory` インターフェースやDI設定が少し煩雑
- Assisted Injectionの理解が必要

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

### 3. `ViewModel` で `SavedStateHandle` から受け取る

`SavedStateHandle.toRoute()` により `route` を取得できます。これにより、Composableを介さずにViewModel側で直接引数を受け取ることができます。

`toRoute()` は内部で `android.os.Bundle` を利用しているため、ユニットテストでは注意が必要です。`android.os.Bundle` はAndroidフレームワークのクラスであり、JVM単体ではテストできません。シミュレートするにはRobolectricが必要になります。また、`savedStateHandle["arg"] = "value"` のように引数を指定する必要があります。

#### ✅ メリット

- `ViewModel` の初期化と同時に取得できる
- 実装がシンプルでコード量が少ない

#### ⚠️ デメリット

- `android.os.Bundle` に依存しておりユニットテストではRobolectricが必要
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

### 4. `SavedStateHandle.toRoute()` を外で呼び、Hiltで注入

テストしやすさを重視する場合、`SavedStateHandle.toRoute()` を切り出して、`ViewModel` には `route` を直接注入する必要があります。

具体的には、`SavedStateHandle.toRoute()` を実行するHilt Moduleを定義します。これにより、`ViewModel` は `route` を直接インジェクトできるため、`ViewModel` が `android.os.Bundle` に依存せず、ユニットテストをより簡単に記述できます。

#### ✅ メリット

- `ViewModel` がAndroidフレームワークに依存しないため、ユニットテストが容易
- DIによる構成がシンプルで、再利用性が高い

#### ⚠️ デメリット

- Hilt Moduleの定義が必要
- Hiltに慣れていないとやや冗長に感じる

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

以下は `ViewModel` での引数受け取りにおける4手法の比較表です。

| 方法               | 初期化タイミング | テストのしやすさ      | 実装の複雑さ          |
| ------------------ | ---------------- | --------------------- | --------------------- |
| 関数呼び出し       | 遅延初期化       | ✅ 初期化を制御できる | 🟡 関数呼び出しが必要 |
| Assisted Injection | 即時初期化       | ✅ DIでテスト容易     | 🟡 Factoryが必要      |
| `SavedStateHandle` | 即時初期化       | ⚠️ Bundleに依存       | 🟢 最小の実装量       |
| Hiltでprovides     | 即時初期化       | ✅ DIでテスト容易     | 🟡 Moduleが必要       |

## おわりに

本記事で紹介した方法は一例であり、特定のベストプラクティスがあるわけではありません。プロジェクトや開発チームに合わせて選択すべきです。今後の変更にも備え、柔軟な選択が求められます。

本記事の方法は現行のNavigation 2.8.0に基づいていますが、Navigation 3のリリースが予定されています。Navigation 3はComposableを主軸に置いており、画面遷移についてが大きな変更があります。本記事で紹介した中では、特に序盤の方法が今後の標準的なアプローチになる可能性が高いと予想しています。

## 参考文献

https://github.com/Daiji256/android-showcase/tree/5eec36161350c0bd5f9a82718abbcb389b48e25a/feature/navigationarguments/src/main/kotlin/io/github/daiji256/showcase/feature/navigationarguments

https://developer.android.com/guide/navigation/design/type-safety

https://developer.android.com/guide/navigation/navigation-3

https://dagger.dev/hilt/

https://dagger.dev/dev-guide/assisted-injection.html
