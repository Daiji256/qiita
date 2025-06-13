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

Navigation Compose 2.8.0 からシリアル化可能なクラスを使うことで、型安全に引数の受け渡しが可能になりました。

詳しくは Android Developers を参照してください。

https://developer.android.com/guide/navigation/design/type-safety

ここでは、引数の受け取り方法に注目し比較します。Navigation Compose 2.8.0 以上と Hilt の利用環境を想定して、サンプルコードを紹介します。うまく応用すればそれ以外の環境でも参考になります。

ここで紹介するコードは、以下のコードを参考に作っています。

https://github.com/Daiji256/android-showcase/tree/5eec36161350c0bd5f9a82718abbcb389b48e25a/feature/navigationarguments/src/main/kotlin/io/github/daiji256/showcase/feature/navigationarguments

## 画面遷移に引数を渡す方法

引数を渡すには、`class` もしくは `data class` で定義されたシリアル化可能な `Route` を利用します。`NavController.navigate()` に `Route` を渡すことで、画面遷移が実行されます。

```Kotlin:TODO 説明などをあとで書く
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

直接的に引数を受け取る方法として、`NavBackStackEntry` を利用する方法があります。`NavGraphBuilder.composable()` の `content` で受け取る `NavBackStackEntry` から画面遷移時に渡した `Route` を受け取ることで、引数受け取りをします。

```Kotlin
fun NavGraphBuilder.sample() {
    composable<SampleRoute> { entry ->
        val route = entry.toRoute<SampleRoute>()
        SampleScreen(route = route)
    }
}
```

### `NavBackStackEntry` からの受け取った `Route` を ViewModel で扱うために

#### Composable から ViewModel のメソッドを呼び出すことで渡す

最も単純な方法として、Composable から ViewModel のメソッドを呼び出す方法があります。以下の例では `LifecycleEventEffect` により `onCreate` のタイミングで ViewModel に `route` を渡しています。しかし、この方法では、`ViewModel` の初期化の後に `route` を取得することになります。`route` が初期化に必要な場合は向いていません。また、`route` を受け取るまでの状態も意識する必要があります。

```Kotlin
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

```Kotlin
@HiltViewModel
class SampleViewModel @Inject constructor() : ViewModel() {
    private val _arg = MutableStateFlow<String?>(null)
    val arg = _arg.asStateFlow()

    fun onCreate(route: SampleRoute) {
        _arg.value = route.arg
    }
}
```

#### ViewModel のインスタンス作成時に渡す

Dagger には Assisted Injection により、一部を Dagger による依存解決、もう一部を手動によって引数を受け取ることができます。それを使うことで、ViewModel の初期化時には `route` を取得できています。

Assisted Injection では Factory を定義し、それを生成される側とする側の両方で扱う必要があります。普段から Assisted Injection を使っている場合は慣れたものだと思いますが、人によってはその手間が大変・難しいと感じるかもしれません。

```Kotlin
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

```Kotlin
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

Composable をかえさず ViewModel 側で引数を受け取ることもできます。`SavedStateHandle.toRoute()` を使うことで、`route` を受け取ることができます。

`SavedStateHandle.toRoute()` は `android.os.Bundle` を利用しているため、注意が必要です。具体的には、そのまま Unit Test を実行できません。Unit Test のためには Robolectric が必要になります。また、`savedStateHandle["arg"] = "dummy"` のように引数を指定する必要があるため、人によっては違和感があるかもしれません。

```Kotlin
@HiltViewModel
class SampleViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
) : ViewModel() {
    private val route = savedStateHandle.toRoute<SampleRoute>()
    val arg = route.arg
}
```

```Kotlin
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

### `Route` を Hilt で provides する

`SavedStateHandle.toRoute()` は `android.os.Bundle` に依存するためテストで扱いづらいという問題がありました。そこで `SavedStateHandle.toRoute()` を ViewModel の外に移し、`route` を直接受け取るようにします。

具体的には、`SavedStateHandle.toRoute()` を行う Module を定義します。これにより、ViewModel は `Route` を直接 Inject できます。そのため、ViewModel が `android.os.Bundle` へ依存しないので、Unit Test を気にせず書くことができます。

しかし、Module を定義することをめんどくさいと感じる人もいるでしょう。

```Kotlin
@Module
@InstallIn(ViewModelComponent::class)
object SampleRouteModule {
    @Provides
    fun providesSampleRoute(savedStateHandle: SavedStateHandle): SampleRoute =
        savedStateHandle.toRoute<SampleRoute>()
}

@HiltViewModel
class SampleViewModel @Inject constructor(
    route: SampleRoute,
) : ViewModel() {
    val arg = route.arg
}
```

## さいごに

ここで紹介した方法が全てではないと思います。また、ベストはありません。プロジェクトや開発者によって選ぶべき手段がかわると思います。

この記事が参考になれば幸いです。

---

Navigation 3 の登場で、大きく変わると思いますが。
