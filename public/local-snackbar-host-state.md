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
ignorePublish: true
---

# LocalSnackbarHostState: CompositionLocal による SnackbarHostState の管理方法

モバイルアプリにおいてユーザーへの情報伝達は非常に重要です。その中でも、短時間でユーザーにフィードバックを伝える際に便利な UI 要素として snackbar があります。

この記事では、Compose における snackbar の基本から、画面などのスコープを意識した snackbar の状態・表示を `CompositionLocal` により上手く扱う方法を紹介します。

## Snackbar とは

Snackbar は、画面の下部に一時的に表示される UI 要素で、ユーザーへの短いメッセージやアクション可能なフィードバックを提供します。

Android には類似の UI 要素として toast があります。Snackbar は toast と比べ以下のような違いがあります。この比較では、表示期間・アクション・デザインといった機能性が注目されることが多いです。この記事では、「スコープ」をより意識します。

Snackbar は表示する要因になった画面やアプリ内で表示されます。そのため、画面遷移等によって表示されなくなる必要があります。Snackbar を扱うとき、どの画面に表示するかは意識する必要がります。

|              | Snackbar                             | Toast              |
| :----------- | :----------------------------------- | :----------------- |
| 表示期間     | 自動消滅・手動消滅                   | 自動消滅           |
| アクション   | ボタンを配置可能                     | 不可               |
| デザイン     | Material Design に準拠、自由度が高い | 変更不可           |
| **スコープ** | **画面・アプリ**                     | **デバイス全体**   |
| 表示例       | [Snackbar のスクショ]                | [Toast のスクショ] |

## Compose における Snackbar

Compose で snackbar を表示するには、`SnackbarHost` と `SnackbarHostState` を使用します。

`SnackbarHost` は snackbar を表示を切り替えるための Composable 関数です。直接配置したり、`Scaffold` 内に配置して利用します。

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

そして、実際に snackbar の表示等の制御を行うのが `SnackbarHostState` です。これは UI の状態を保持するステートホルダーです。

```kotlin
val snackbarHostState = remember { SnackbarHostState() }
```

Snackbar を表示するには、`SnackbarHostState` の `showSnackbar` メソッドを呼び出します。このメソッドは suspend 関数であるため、コルーチン内で呼び出す必要があります。

```kotlin
val coroutineScope = rememberCoroutineScope()
Button(
    onClick = {
        coroutineScope.launch {
            snackbarHostState.showSnackbar(message = "message")
        }
    },
) {
    Text(text = "show snackbar")
}
```

## `CompositionLocal` とは

Snackbar は画面等のスコープを意識して表示を切り替える必要があります。そのため、画面ごとに `SnackbarHost` と `SnackbarHostState` が必要になります。ブラウザを開けない時の特定のエラーや、子・孫コンポーネント内から snackbar を表示したい場合、`SnackbarHostState` の扱いが大変です。画面の最初に `SnackbarHostState` を宣言して、子に渡す必要があります。

そこで、`CompositionLocal` を利用します。`CompositionLocal` は、Composition を通じてデータを暗黙的に渡すための仕組みです。通常の引数によるデータ渡しとは異なり、階層の深い部分にある Composable でも、親から明示的に引数として渡されなくても、特定の `CompositionLocal` で提供された値にアクセスできるようになります。

これにより、「この画面ではこの `SnackbarHostState` を利用する」のようなスコープを扱うことができます。子コンポーネントでは、自動的にスコープに合わせた `SnackbarHostState` が利用できます。

`CompositionLocal` を使って snackbar を管理する具体的な例を紹介します。`SnackbarHostState` を `CompositionLocal` として提供することで、必要な場所で簡単に `SnackbarHostState` にアクセスできるようになります。

まず、`SnackbarHostState` を保持する `CompositionLocal` を定義します。

```kotlin
val LocalSnackbarHostState = staticCompositionLocalOf { SnackbarHostState() }
```

### `Scaffold` に対して

`Scaffold` を含む Composable で `LocalSnackbarHostState` を提供し、その下位の Composable から `SnackbarHostState` を利用できるようにします。

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

これで、`MyScaffold` 以下のどの Composable からでも、以下のように `LocalSnackbarHostState.current` を使って `SnackbarHostState` を取得し、その画面に snackbar を表示できるようになります。

```kotlin
@Composable
fun Something() {
    val snackbarHostState = LocalSnackbarHostState.current
    val coroutineScope = rememberCoroutineScope()
    Button(
        onClick = {
            coroutineScope.launch {
                snackbarHostState.showSnackbar(message = "message")
            }
        },
    ) {
        Text(text = "show snackbar")
    }
}
```

### `NavGraphBuilder.composable` に対して

`Scaffold` を利用しない場合や、`Scaffold` よりも広い範囲でスナックバーのスコープを管理したい場合は `NavGraphBuilder.composable` にて `SnackbarHostState` を提供すると良いでしょう。この場合も同様に`CompositionLocalProvider`を利用できます。

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

TODO
以下のような独自の何かを実行する機能を実装したとして、それで簡単にスナックバーを表示できるという話を書きたい。

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

`CompositionLocal` を使って `SnackbarHostState` を管理することにはいくつかのメリットがあります。

- スコープ範囲: 子コンポーネントがそのスコープの snackbar の表示ができる
- コードの集約: `SnackbarHostState` を扱うためのコードが 1 箇所になり、他コンポーネントは描画に専念しやすい
- バケツリレーの解消: `SnackbarHostState` を引数で渡す必要がない

## まとめ

この記事では、Compose における snackbar の基本から、`SnackbarHostState` を `CompositionLocal` を使って管理する方法について紹介しました。

`CompositionLocal` を使うことで、snackbar のスコープを上手く扱うことだできます。

## 参考文献

https://m3.material.io/components/snackbar/overview

https://developer.android.com/guide/topics/ui/notifiers/toasts

https://developer.android.com/develop/ui/compose/components/snackbar

https://developer.android.com/develop/ui/compose/compositionlocal
