---
title: Jetpack Composeで和欧混植（文字単位のフォントフォールバック）を実現する
tags:
  - Android
  - Kotlin
  - font
  - Compose
  - JetpackCompose
private: false
updated_at: '2026-07-04T08:40:09+09:00'
id: 329b4325977089ba7383
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

タイポグラフィを考えるとき「アルファベットはフォントAを、日本語はフォントBを表示したい」という和欧混植の要望もあるでしょう。Jetpack Composeで開発しているAndroidアプリでも、この要望に応えたい場面があります。

しかし、`FontFamily(Font(欧文), Font(和文))` のようにいくつかの `Font` をリスト形式で指定するだけでは、文字（グリフ）単位できれいにフォールバックされず、意図した混植にはなりません。

この記事では、AndroidネイティブのAPIを活用し、Compose上で文字単位のフォールバックを実現する方法を紹介します。

## `Typeface.CustomFallbackBuilder` と `AndroidFont`

Android 10（API 29）で追加された `Typeface.CustomFallbackBuilder` は、「最初のフォントに文字がなければ次のフォントを試す」という文字単位のフォールバックを実現できるAPIです。

これをJetpack Composeの `AndroidFont` で扱うことで、文字単位でフォールバックする `FontFamily` を構築できます。

## 実装例

ここでは、欧文に「Quicksand」、和文に「Zen Maru Gothic」を表示するための実装を紹介します。バリアブルフォント（可変フォント）と静的フォントの双方に対応した実装としています。万が一の未対応文字にはシステム標準フォントを割り当てています。

```kotlin
// ...
import android.graphics.Typeface as NativeTypeface
import android.graphics.fonts.Font as NativeFont
import android.graphics.fonts.FontFamily as NativeFontFamily

@RequiresApi(Build.VERSION_CODES.Q)
val MyFontFamily = FontFamily(
    fonts = listOf(
        MyFont(weight = FontWeight.Normal),
        MyFont(weight = FontWeight.Bold),
    ),
)

@RequiresApi(Build.VERSION_CODES.Q)
private class MyFont(
    override val weight: FontWeight,
) : AndroidFont(
    loadingStrategy = FontLoadingStrategy.Blocking,
    typefaceLoader = MyFontTypefaceLoader,
    variationSettings = FontVariation.Settings(),
) {
    override val style: FontStyle = FontStyle.Normal

    private var typeface: NativeTypeface? = null

    private fun load(context: Context): NativeTypeface {
        val quicksandFont = NativeFont.Builder(
            context.resources,
            R.font.quicksand,
        )
            .setWeight(weight.weight)
            .setFontVariationSettings("'wght' ${weight.weight}")
            .build()
        val quicksandFontFamily = NativeFontFamily.Builder(quicksandFont).build()

        val zenMaruGothicFont = NativeFont.Builder(
            context.resources,
            when {
                weight <= FontWeight.Normal -> R.font.zen_maru_gothic_regular
                else -> R.font.zen_maru_gothic_bold
            },
        )
            .setWeight(weight.weight)
            .build()
        val zenMaruGothicFontFamily = NativeFontFamily.Builder(zenMaruGothicFont).build()

        return NativeTypeface.CustomFallbackBuilder(quicksandFontFamily)
            .addCustomFallback(zenMaruGothicFontFamily)
            .setSystemFallback("sans-serif")
            .build()
            .also { typeface = it }
    }

    private object MyFontTypefaceLoader : TypefaceLoader {
        override fun loadBlocking(context: Context, font: AndroidFont): NativeTypeface? =
            (font as? MyFont)?.run { typeface ?: load(context) }

        override suspend fun awaitLoad(context: Context, font: AndroidFont): Nothing =
            throw UnsupportedOperationException("MyFont is blocking")
    }
}
```

## まとめ

- 単に `FontFamily` へ複数の `Font` を指定しても、文字単位のフォールバックは実現できない
- Android 10（API 29）で追加された `Typeface.CustomFallbackBuilder` を使えば、文字単位のフォールバックが実現できる
- `Typeface` を `AndroidFont` でラップすることで、フォールバックの機能をComposeでも利用できる

## 参考文献

- [Typeface.CustomFallbackBuilder | Android Developers](https://developer.android.com/reference/kotlin/android/graphics/Typeface.CustomFallbackBuilder)
- [AndroidFont | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/ui/text/font/AndroidFont)
- [Google Fonts](https://fonts.google.com/)
