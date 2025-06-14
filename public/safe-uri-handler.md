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

`UriHandler` とは `androidx.compose.ui.platform` で提供される URI を開くための機能です。`UriHandler` を使うことで URI を簡単に開くことができます：

```kotlin
uriHandler.openUri(uri)
```

以下のように、`LocalUriHandler` により `UriHandler` を取得できます：

```kotlin
val uriHandler = LocalUriHandler.current
```

## `AndroidUriHandler` について

Android ではデフォルトで `AndroidUriHandler` が使われています。

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

## `ActivityNotFoundException` を握りつぶしたい

`UriHandler` の使い所として

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

## 参考文献

https://github.com/Daiji256/android-showcase/blob/dcbfb7ac151b09a6995ece9f2081c23e417842e8/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/urihandler/SafeUriHandler.kt

https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/UriHandler

https://cs.android.com/search?q=file:androidx/compose/ui/platform/AndroidUriHandler.android.kt+class:androidx.compose.ui.platform.AndroidUriHandler
