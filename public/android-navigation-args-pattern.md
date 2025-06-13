---
title: Navigation Compose での引数の受け取り方を比較
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

## はじめに

Navigation Compose 2.8.0 以降では、シリアル化可能なクラスを用いて、型安全に画面間で引数を受け渡すことが可能になりました。

本記事では、特に**引数の受け取り方法**に焦点を当て、複数の実装方法を比較検討します。Navigation Compose 2.8.0 以降と Hilt の利用環境を想定したサンプルコードを提示しますが、これらを応用することでその他の環境でも参考になるかと思います。

Navigation Compose 2.8.0 での画面遷移の詳細については、Android Developers の公式ドキュメントをご参照ください。

https://developer.android.com/guide/navigation/design/type-safety

ここで紹介するコードは、以下のリポジトリのコードを元に作成しています。

https://github.com/Daiji256/android-showcase/tree/5eec36161350c0bd5f9a82718abbcb389b48e25a/feature/navigationarguments/src/main/kotlin/io/github/daiji256/showcase/feature/navigationarguments

## 画面遷移に引数を渡す方法

引数を渡すには、`class` もしくは `data class` で定義されたシリアル化可能な `Route` クラスを利用します。`route` を `NavController.navigate()` メソッドに渡すことで、画面遷移を実行します。

```kotlin
@Serializable
data class SampleRoute(val arg: String)

fun NavGraphBuilder.sample() {
    composable<SampleRoute> {
        SampleScreen(/* ... */)
    }
}

fun NavController.navigateToSample(arg: String) =
    navigate(route = SampleRoute(arg = arg))
```

## `NavBackStackEntry` からの受け取り

直接的に引数を受け取る方法として、`NavBackStackEntry` を利用する方法があります。`NavGraphBuilder.composable()` の `content` ラムダで受け取る `NavBackStackEntry` から、画面遷移時に渡された `route` を抽出することで引数を受け取ります。

```kotlin
fun NavGraphBuilder.sample() {
    composable<SampleRoute> { entry ->
        val route = entry.toRoute<SampleRoute>()
        SampleScreen(route = route)
    }
}
```

### `route` を ViewModel に渡す方法

#### Composable から ViewModel のメソッドを呼び出すことで渡す

最も単純な方法として、Composable から ViewModel のメソッドを呼び出す方法があります。以下の例では `LifecycleEventEffect` を利用し、`ON_CREATE` イベントのタイミングで ViewModel に `route` を渡しています。しかし、この方法では `ViewModel` の初期化後に `route` を取得することになるため、`route` が初期化に必須な場合には適していません。また、`route` を受け取るまでの ViewModel の状態も考慮する必要があります。

一方で、Composable 側で初期化のタイミングやライフサイクルを制御したい場合には適している方法です。

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

#### ViewModel のインスタンス作成時に渡す (Assisted Injection)

Dagger の Assisted Injection を利用することで、ViewModel のコンストラクタで一部の依存関係を Dagger による依存解決に任せつつ、別の引数を手動で受け取ることが可能です。これにより、ViewModel の初期化時に `route` を取得することができます。

Assisted Injection を利用するには、Factory インターフェースを定義し、それを ViewModel と Composable の両方で扱う必要があります。普段から Assisted Injection を利用している開発者にとっては慣れた手法ですが、そうでない場合はその設定に手間や難しさを感じるかもしれません。

この方法は、`SavedStateHandle` も介さずテストも用意です。また、引数の流れが明確になるため、設計上も理解しやすい手法です。

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

### ViewModel で `SavedStateHandle` から受け取る

Composable を介さずに ViewModel 側で直接引数を受け取ることも可能です。`SavedStateHandle.toRoute()` を使用することで、`route` を取得できます。

`SavedStateHandle.toRoute()` は内部で `android.os.Bundle` を利用しているため、ユニットテストでは注意が必要です。`android.os.Bundle` は Android フレームワークのクラスであり、JVM 単体ではテストできません。JVM 上でシミュレートするために Robolectric が必要になります。また、`savedStateHandle["arg"] = "dummy"` のように引数を指定する必要があるため、人によっては違和感を持つかもしれません。

今回紹介する方法の中では、最もコード量が少ない実装です。実装時の作業コストを懸念する方には適しています。

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

### `route` を Hilt で provides する

`SavedStateHandle.toRoute()` が `android.os.Bundle` に依存するためテストで扱いにくいという問題を解決する方法として、`SavedStateHandle.toRoute()` の呼び出しを ViewModel の外に移し、`route` を ViewModel に直接注入する方法が挙げられます。

具体的には、`SavedStateHandle.toRoute()` を実行する Hilt Module を定義します。これにより、ViewModel は `Route` を直接インジェクトできるため、ViewModel が `android.os.Bundle` に依存せず、ユニットテストをより簡単に記述できるようになります。

このために Module を定義する手間が気になる方もいるかもしれません。一方で、Hilt の扱いに慣れた開発者にとっては一番単純な方法とも言えます。

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

## さいごに

ここで紹介した方法は一例であり、特定のベストプラクティスがあるわけではありません。プロジェクトの要件や開発チームの慣習によって、最適な方法は異なります。

また、今後リリースが予定されている Navigation 3 では画面遷移が大きく変更されます。Navigation 3 は Composable を主軸に置いています。本記事で紹介した中では、特に序盤の方法が今後の標準的なアプローチになる可能性があります。

この記事が、Navigation Compose での引数受け取り方法を検討するときの参考になれば幸いです。
