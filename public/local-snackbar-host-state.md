---
title: "LocalSnackbarHostState: CompositionLocal による SnackbarHostState の管理方法"
tags:
  - Android
  - Kotlin
  - JetpackCompose
  - Snackbar
  - CompositionLocal
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

モバイルアプリにおいてユーザーへの情報伝達は非常に重要です。その中でも、短時間でユーザーにフィードバックを伝える際に便利な UI 要素として Snackbar があります。

この記事では、Compose における Snackbar の基本から、画面などのスコープを意識した Snackbar の状態・表示を `CompositionLocal` によりうまく扱う方法を紹介します。

## Snackbar とは

Snackbar は、画面の下部に一時的に表示される UI 要素で、ユーザーへの短いメッセージやアクション可能なフィードバックを提供します。

Android には類似の UI 要素として Toast があります。Snackbar は Toast と以下のような点で異なります。表示期間・アクション・デザインといった機能性が注目されることが多いですが、この記事では「スコープ」に着目します。

Snackbar は表示する要因になった画面やアプリ内で表示されます。そのため、画面遷移などによって表示されなくなる必要があります。Snackbar を扱うとき、どの画面に表示するかは意識する必要があります。

|              | Snackbar                                                                                                                     | Toast                                                                                                                     |
| :----------- | :--------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------ |
| 表示期間     | 自動消滅・手動消滅                                                                                                           | 自動消滅                                                                                                                  |
| アクション   | ボタンを配置可能                                                                                                             | 不可                                                                                                                      |
| デザイン     | Material Design に準拠、自由度が高い                                                                                         | 変更不可                                                                                                                  |
| **スコープ** | **画面・アプリ**（特定機能に対して）                                                                                         | **デバイス全体**（システムに対して）                                                                                      |
| 表示例       | ![snackbar.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/93d731bc-e707-419f-9288-efc93cd492d3.gif) | ![toast.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/995b4127-1ef8-44ab-aadb-c884e3a7f043.gif) |

## Compose における Snackbar

Compose で Snackbar を表示するには、`SnackbarHost` と `SnackbarHostState` を使用します。

`SnackbarHost` は Snackbar の表示領域を定義する Composable 関数です。通常、`Scaffold`（Material Design による基本的な画面レイアウト）の `snackbarHost` に配置して利用します。これにより、Snackbar が画面の適切な位置に表示されるようになります。

```kotlin
Scaffold(
    snackbarHost = {
        SnackbarHost(
            hostState = snackbarHostState,
        )
    },
    // ...
)
```

そして、実際に Snackbar の表示・非表示やメッセージ内容などの制御を行うのが `SnackbarHostState` です。これは UI の状態を保持するステートホルダーであり、Composable 関数から利用する場合は `remember` を使って状態を記憶させます。

```kotlin
val snackbarHostState = remember { SnackbarHostState() }
```

Snackbar を表示するには、`SnackbarHostState` の `showSnackbar` を呼び出します。このメソッドは suspend 関数であるため、コルーチン内で呼び出す必要があります。

```kotlin
val coroutineScope = rememberCoroutineScope()
Button(
    onClick = {
        coroutineScope.launch {
            snackbarHostState.showSnackbar(message = "message")
        }
    },
) {
    Text(text = "Show Snackbar")
}
```

## `CompositionLocal` とは

Snackbar は画面などの特定のスコープを意識して表示を切り替える必要があります。そのため、画面ごとに `SnackbarHost` と `SnackbarHostState` が必要になるのが一般的です。しかし、アプリ内で深くネストされた子・孫コンポーネントから Snackbar を表示したい場合、`SnackbarHostState` を親コンポーネントから引数として順々に渡していくバケツリレー（Props Drilling）が発生し、コードの保守性が低下することがあります。

そこで、`CompositionLocal` が役立ちます。`CompositionLocal` は、Compose の composition を通じてデータやサービスを暗黙的に「渡す」ための仕組みです。通常の引数によるデータ渡しとは異なり、階層の深い部分にある Composable 関数でも、親から明示的に引数として渡されなくても、特定の `CompositionLocal` で提供された値にアクセスできるようになります。

これにより、「この画面ではこの `SnackbarHostState` を利用する」といった、スコープに紐づいた Snackbar の管理が可能になります。子コンポーネントは、自動的にそのスコープに合わせた `SnackbarHostState` を利用できるため、コードがよりすっきりと記述できます。

`CompositionLocal` を使って Snackbar を管理する具体的な例を紹介します。まず、`SnackbarHostState` を保持する `CompositionLocal` を定義します。

```kotlin
val LocalSnackbarHostState = staticCompositionLocalOf { SnackbarHostState() }
```

### `Scaffold` に対して

`Scaffold` を含む Composable 関数で `LocalSnackbarHostState` を提供し、その下位の Composable 関数から `SnackbarHostState` を利用できるようにします。

このように `MyScaffold` を実装しておけば、それ以下の Composable 関数からでも `LocalSnackbarHostState.current` を使って `SnackbarHostState` を取得し、その画面に Snackbar を表示できます。つまり、この `Something` は、`MyScaffold` 内に配置されていれば、`MyScaffold` に Snackbar を表示します。

```kotlin
@Composable
fun MyScaffold(
    snackbarHostState: SnackbarHostState = remember { SnackbarHostState() },
    // ...
) {
    CompositionLocalProvider(
        LocalSnackbarHostState provides snackbarHostState,
    ) {
        Scaffold(
            snackbarHost = {
                SnackbarHost(hostState = snackbarHostState)
            },
            // ...
        )
    }
}
```

```kotlin
@Composable
fun Something() {
    val snackbarHostState = LocalSnackbarHostState.current
    val coroutineScope = rememberCoroutineScope()
    Button(
        onClick = {
            coroutineScope.launch {
                snackbarHostState.showSnackbar(message = "Message")
            }
        },
    ) {
        Text(text = "Show Snackbar")
    }
}
```

### `NavGraphBuilder.composable` に対して

`Scaffold` を利用しない場合や、`Scaffold` よりも広い範囲で Snackbar のスコープを管理したい場合は、ナビゲーショングラフ定義時に `SnackbarHostState` を提供すると良いでしょう。この場合も同様に `CompositionLocalProvider` を利用できます。これにより、特定の画面（ルート）に遷移したときに、その画面でのみ有効な `SnackbarHostState` が `CompositionLocal` として提供されます。

このように `NavGraphBuilder.myComposable` を実装した場合、`LocalSnackbarHostState.current` を使って同様の `SnackbarHostState` を取得できます。

```kotlin
inline fun <reified T : Any> NavGraphBuilder.myComposable(
    // ...
    noinline content: @Composable AnimatedContentScope.(NavBackStackEntry) -> Unit,
) = composable<T>(/* ... */) { entry ->
    CompositionLocalProvider(
        LocalSnackbarHostState provides remember { SnackbarHostState() },
    ) {
        content(entry)
    }
}
```

## 他にも便利なこと

`CompositionLocal` を使うと、コンポーネントから直接 Snackbar を操作できるようになるだけでなく、よりアプリケーション全体で利用可能な共通ロジックを実装する際に便利です。例えば、以下のように URI を開く処理でエラーが発生した場合に Snackbar を表示する独自の `UriHandler` を定義できます。

この `rememberMyUriHandler` は、`LocalSnackbarHostState` に依存しています。もし、`LocalSnackbarHostState` が `CompositionLocal` として提供されていなければ、これを呼び出すたびに `SnackbarHostState` を引数で渡す必要があり、使い勝手が悪いです。

```kotlin
@Composable
fun rememberMyUriHandler(): UriHandler {
    val uriHandler = LocalUriHandler.current
    val snackbarHostState = LocalSnackbarHostState.current
    val coroutineScope = rememberCoroutineScope()
    val onFailure: State<(uri: String) -> Unit> = rememberUpdatedState { uri ->
        coroutineScope.launch {
            snackbarHostState.showSnackbar(message = "Can't open $uri.")
        }
    }
    return remember(uriHandler, onFailure) {
        object : UriHandler {
            override fun openUri(uri: String) {
                try {
                    uriHandler.openUri(uri)
                } catch (e: IllegalArgumentException) {
                    onFailure.value(uri)
                }
            }
        }
    }
}
```

## まとめ

この記事では、Compose における Snackbar の基本から、`SnackbarHostState` を `CompositionLocal` を使って管理する方法について紹介しました。`CompositionLocal` を使うことで、Snackbar のスコープをうまく扱うことができ、以下のようなメリットがあります。

- スコープ範囲の明確化: 子コンポーネントが、そのスコープに紐づく Snackbar を表示できるようになる
- コードの集約: `SnackbarHostState` を扱うためのコードが 1 箇所に集約され、他のコンポーネントは表示のための実装に専念しやすくなる
- バケツリレーの解消: `SnackbarHostState` を引数で渡す必要がなくなるため、コードの可読性と保守性が向上する

## 参考文献

https://m3.material.io/components/snackbar/overview

https://developer.android.com/guide/topics/ui/notifiers/toasts

https://developer.android.com/develop/ui/compose/components/snackbar

https://developer.android.com/develop/ui/compose/compositionlocal
