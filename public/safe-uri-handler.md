---
title: 簡単に安全に UriHandler を使う方法
tags:
  - Android
  - Kotlin
  - JetpackCompose
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# 簡単に安全に UriHandler を使う方法

## `UriHandler` とは

`UriHandler` は `androidx.compose.ui.platform` が提供する、Compose 上から URI（Web ページ、ファイル、外部アプリなど）を開くためのインターフェースです。Compose 内では例えばこんなふうにシンプルに呼び出せます：

```kotlin
@Composable
fun OpenUriButton(uri: String) {
    val uriHandler = LocalUriHandler.current
    Button(onClick = { uriHandler.openUri(uri) }) {
        Text(text = "Open URI")
    }
}
```

## デフォルトの `UriHandler`：`AndroidUriHandler`

`LocalUriHandler.current` で取得できるのは、プラットフォームごとに実装された `UriHandler` です。Android 環境ではデフォルトで `AndroidUriHandler` が使われます：

```kotlin
class AndroidUriHandler(private val context: Context) : UriHandler {
    override fun openUri(uri: String) {
        try {
            context.startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(uri)))
        } catch (e: ActivityNotFoundException) {
            throw IllegalArgumentException("Can't open $uri.", e)
        }
    }
}
```

実装から分かる通り、

- URI に対応アプリがインストールされていないとき
- 無効な URI の形式のとき

などでは `IllegalArgumentException` が throw されます。そのため、呼び出し側ではクラッシュを防ぐために例外処理を用意する必要があります。

## 簡単に安全に URI を開くために

記事中のリンクやお知らせなど、ユーザーに URI を開かせたいシーンは多いものの、毎回例外処理を呼び出し側で実装するのは手間です。加えて、ライブラリや SDK 内で `UriHandler` を呼び出している場合は、例外処理を差し込むのが難しいです。

そこで、例外の代わりにトーストを表示する独自の `UriHandler`（`SafeUriHandler`）を実装します：

```kotlin
class SafeUriHandler(private val context: Context) : UriHandler {
    override fun openUri(uri: String) {
        try {
            context.startActivity(Intent(Intent.ACTION_VIEW, uri.toUri()))
        } catch (e: ActivityNotFoundException) {
            Toast.makeText(context, "Can't open $uri.", Toast.LENGTH_SHORT).show()
        }
    }
}
```

`CompositionLocalProvider` で `LocalUriHandler` 上書きすることで、以降は `LocalUriHandler.current` 経由で `SafeUriHandler` を取得できます：

```kotlin
@Composable
fun MyApp() {
    val context = LocalContext.current
    val uriHandler = remember(context) { SafeUriHandler(context) }
    CompositionLocalProvider(
        LocalUriHandler provides uriHandler,
    ) {
        // ...
    }
}
```

## まとめ

- `LocalUriHandler.current` でプラットフォーム標準の `UriHandler` を取得できる
- デフォルトの `AndroidUriHandler` は起動失敗時に例外を throw する
- `SafeUriHandler` のように例外を throw しないことで、呼び出し側での例外処理を不要にできる
- `CompositionLocalProvider` で `LocalUriHandler` を上書きすることでアプリ全体で独自の URI オープン処理を一括適用できる

## 参考文献

https://github.com/Daiji256/android-showcase/blob/dcbfb7ac151b09a6995ece9f2081c23e417842e8/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/urihandler/SafeUriHandler.kt

https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/UriHandler

https://cs.android.com/search?q=file:androidx/compose/ui/platform/AndroidUriHandler.android.kt+class:androidx.compose.ui.platform.AndroidUriHandler
