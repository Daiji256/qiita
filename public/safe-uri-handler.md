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

## `UriHandler` とは

`UriHandler` は `androidx.compose.ui.platform` が提供する、URI（Web ページ、ファイル、外部アプリなど）を開くためのインターフェースです。Composable 関数からは、以下のようにシンプルに呼び出せます。

```kotlin
@Composable
fun OpenUriButton(uri: String) {
    val uriHandler = LocalUriHandler.current
    Button(onClick = { uriHandler.openUri(uri) }) {
        Text(text = "Open URI")
    }
}
```

## Android におけるデフォルトの `UriHandler`

`LocalUriHandler.current` で取得されるのは、各プラットフォームに応じた `UriHandler` の実装です。Android 環境では、デフォルトとして `AndroidUriHandler` が使用されます。

`AndroidUriHandler` では、URI に対応するアプリがインストールされていない場合や、URI の形式が無効な場合などに `IllegalArgumentException` が throw されます。そのため、呼び出し側ではクラッシュ（アプリの異常終了）を防ぐため、例外処理を行う必要があります。

<details><summary>AndroidUriHandler.android.kt 全文</summary>

https://cs.android.com/search?q=file:androidx/compose/ui/platform/AndroidUriHandler.android.kt+class:androidx.compose.ui.platform.AndroidUriHandler

```kotlin
/*
 * Copyright 2020 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package androidx.compose.ui.platform

import android.content.ActivityNotFoundException
import android.content.Context
import android.content.Intent
import android.net.Uri

class AndroidUriHandler(private val context: Context) : UriHandler {

    /**
     * Open given URL in browser
     *
     * @throws IllegalArgumentException when given [uri] is invalid and/or can't be handled by the
     *   system
     */
    override fun openUri(uri: String) {
        try {
            context.startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(uri)))
        } catch (e: ActivityNotFoundException) {
            throw IllegalArgumentException("Can't open $uri.", e)
        }
    }
}

```

</details>

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

しかし、毎回呼び出し側で例外処理を実装するのは煩雑です。また、ライブラリや SDK 内で `UriHandler` を呼び出している場合は、例外処理を差し込むのが難しいケースもあります。

そこで、例外を throw する代わりに Toast を表示する独自の `UriHandler`（`SafeUriHandler`）を実装しました[^toast]。`CompositionLocalProvider` を使用して `LocalUriHandler` を上書きすることで、以降の処理では `LocalUriHandler.current` を通じて `SafeUriHandler` を取得できます。これにより、アプリケーション全体で安全な URI オープン処理を適用することが可能です。

[^toast]: `SafeUriHandler` は簡易な通知手段として Toast を利用していますが、UI の状態に応じて Snackbar や Dialog を用いる方が適切な場合もあります。

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

|                                                          URI を開ける場合                                                           |                                                         URI を開けない場合                                                          |
| :---------------------------------------------------------------------------------------------------------------------------------: | :---------------------------------------------------------------------------------------------------------------------------------: |
| <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/f1d26180-d8dc-4494-9fc1-a5b03b8c8ba6.gif" width="180"> | <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/4b17e779-47d0-4cd5-a5fb-59bd5bfcaeb0.gif" width="180"> |

## まとめ

- `LocalUriHandler.current` を使用すると、プラットフォーム標準の `UriHandler` を取得できます
- デフォルトの `AndroidUriHandler` は、URI の起動に失敗した際に例外を throw し、アプリがクラッシュする可能性があります
- `SafeUriHandler` のように例外を throw しない独自の `UriHandler` を実装することで、呼び出し側での例外処理が不要になります
- `CompositionLocalProvider` を使って `LocalUriHandler` を上書きすることで、アプリ全体で独自処理を一括して適用できます

## 参考文献

https://github.com/Daiji256/android-showcase/blob/dcbfb7ac151b09a6995ece9f2081c23e417842e8/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/urihandler/SafeUriHandler.kt

https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/UriHandler

https://cs.android.com/search?q=file:androidx/compose/ui/platform/AndroidUriHandler.android.kt+class:androidx.compose.ui.platform.AndroidUriHandler
