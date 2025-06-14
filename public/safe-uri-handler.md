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

`UriHandler` は `androidx.compose.ui.platform` が提供する、アプリ内から URI（ウェブページ、他アプリの画面、ファイルなど）を開くためのインターフェースです。Compose の中で次のようにシンプルに呼び出せます：

```kotlin
val uriHandler = LocalUriHandler.current
Button(
    onClick = { uriHandler.openUri("https://example.com") },
) {
    Text(text = "open uri")
}
```

`LocalUriHandler.current` から取得できるのが、プラットフォームごとに実装された `UriHandler` です。

## デフォルトの `AndroidUriHandler`

Android 環境では、デフォルトでは `AndroidUriHandler` が使われています：

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

ブラウザ等の URI を開くことができるアプリが無いときや、URI が無効なときに `IllegalArgumentException` が throw されるます。そのため、呼び出しのさいは失敗を考慮する必要があります。

## 簡単に安全に URI を開くために

`UriHandler` を利用する主要な目的として、記事やお知らせなどからホームページ等のウェブサイトなどを開くことがあげられます。これらの場合、起動に失敗した場合ユーザーにそのことを通知すれば十分なこととが多いです。

しかし、呼び出しのたびに例外の対応をするのは大変です。また、ライブラリや SDK に隠れていて、例外時の処理が難しいこともあります。

そこで、例外を throw する代わりに Toast を表示する `UriHandler` を実装します。`UriHandler` は interface のため、次のように独自実装できます：

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

`SafeUriHandler` を `CompositionLocalProvider` で `LocalUriHandler` に provides することで、それ以下で例外の throw を気にせず `UriHandler` を使うことができます：

```kotlin
val context = LocalContext.current
val uriHandler = remember(context) { SafeUriHandler(context) }
CompositionLocalProvider(
    LocalUriHandler provides uriHandler,
) { /* ... */ }
```

## まとめ

- `LocalUriHandler.current` で `UriHandler` を取得できる
- 例外を throw する代わりに Toast を 表示する `UriHandler` を実装した
- `LocalUriHandler` を上書きすることで、アプリ全体で気軽に安全に利用できる

## 参考文献

https://github.com/Daiji256/android-showcase/blob/dcbfb7ac151b09a6995ece9f2081c23e417842e8/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/urihandler/SafeUriHandler.kt

https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/UriHandler

https://cs.android.com/search?q=file:androidx/compose/ui/platform/AndroidUriHandler.android.kt+class:androidx.compose.ui.platform.AndroidUriHandler
