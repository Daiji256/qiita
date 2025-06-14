---
title: 簡単で安全な UriHandler の実装例と利用方法
tags:
  - Android
  - Kotlin
  - JetpackCompose
  - Intent
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# 簡単で安全な UriHandler の実装例と利用方法

## `UriHandler` とは

`UriHandler` は `androidx.compose.ui.platform` が提供する、Compose 上から URI（Web ページ、ファイル、外部アプリなど）を開くためのインターフェースです。Composable 関数からは以下のようにシンプルに呼び出すことができます。

```kotlin
@Composable
fun OpenUriButton(uri: String) {
    val uriHandler = LocalUriHandler.current
    Button(onClick = { uriHandler.openUri(uri) }) {
        Text(text = "Open URI")
    }
}
```

## Android でのデフォルトの `UriHandler`

`LocalUriHandler.current` で取得できるのは、プラットフォームごとに実装された `UriHandler` です。Android 環境では、デフォルトで `AndroidUriHandler` が使用されます。

実装から分かる通り、URI に対応するアプリがインストールされていない場合や無効な URI の場合などでは `IllegalArgumentException` が throw されます。そのため、呼び出し側ではクラッシュ（アプリの異常終了）を防ぐために例外処理を行う必要があります。

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

## 簡単に安全に URI を開くために

記事中のリンクやお知らせなど、ユーザーに URI を開かせたいシーンは多々あります。ほとんどの場合では URI を開くのに失敗したときは、ユーザーにそのことを伝えれば十分です。

そのために、毎回呼び出し側で例外処理を実装するのは手間です。また、ライブラリや SDK 内で `UriHandler` を呼び出している場合は、例外処理を差し込むのが難しいケースもあります。

そこで、例外を throw する代わりにトーストを表示する独自の `UriHandler`（`SafeUriHandler`）を実装してみました。`CompositionLocalProvider` を使用して `LocalUriHandler` を上書きすることで、以降の処理では `LocalUriHandler.current` 経由で `SafeUriHandler` を取得できます。これにより、アプリケーション全体で安全な URI オープン処理を適用することが可能です。

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

[SafeUriHandler のデモ動画](TODO)

## まとめ

- `LocalUriHandler.current` を使用すると、プラットフォーム標準の `UriHandler` を取得できます
- デフォルトの `AndroidUriHandler` は、URI の起動に失敗した際に例外を throw し、アプリがクラッシュする可能性があります
- `SafeUriHandler` のように例外を throw しない独自の `UriHandler` を実装することで、呼び出し側での例外処理を不要にできます
- `CompositionLocalProvider` を使って `LocalUriHandler` を上書きすることで、アプリ全体で独自処理を一括して適用できます

## 参考文献

https://github.com/Daiji256/android-showcase/blob/dcbfb7ac151b09a6995ece9f2081c23e417842e8/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/urihandler/SafeUriHandler.kt

https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/UriHandler

https://cs.android.com/search?q=file:androidx/compose/ui/platform/AndroidUriHandler.android.kt+class:androidx.compose.ui.platform.AndroidUriHandler
