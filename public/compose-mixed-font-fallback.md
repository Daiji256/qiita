---
title: Jetpack Composeで和欧混植（文字単位のフォントフォールバック）を実現する
tags:
  - Android
  - Kotlin
  - JetpackCompose
  - Compose
  - Font
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

タイポグラフィを考えるとき「アルファベットはフォントAを、日本語はフォントBを表示したい」という和欧混植の要望があることもあるでしょう。Jetpack Composeにより開発しているAndroidアプリにおいても、この要望を叶えたいことがあります。

`FontFamily(Font(欧文), Font(和文))` のようにいくつかの `Font` をリスト形式で指定したところで、文字単位でフォールバックされず意図した混植にはなりません。

この記事では、AndroidネイティブのAPIを活用し、Compose上で文字単位のフォールバックを実現する方法を紹介します。

## `Typeface.CustomFallbackBuilder` と `AndroidFont`

Android 10（API 29）で追加された `Typeface.CustomFallbackBuilder` は、「最初のフォントに文字がなければ次のフォントを試す」という文字単位のフォールバック実現できるAPIです。

これをJetpack Composeの `AndroidFont` に変換することで、文字単位でフォールバックする `Font` / `FontFamily` を構築できます。

## 実装例

ここでは、欧文に「Quicksand」、和文に「Zen Maru Gothic」を表示するための実装を紹介します。万が一の未対応文字にはシステム標準フォントを割り当てています。

実装例から分かるとおり、バリアブルフォント・非バリアブルフォントともに対応しています。

```kotlin
// ...
import android.graphics.Typeface as NativeTypeface
import android.graphics.fonts.Font as NativeFont
import android.graphics.fonts.FontFamily as NativeFontFamily

val MyFontFamily = FontFamily(
    fonts = listOf(
        MyFont(weight = FontWeight.Normal),
        MyFont(weight = FontWeight.Bold),
    ),
)

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

- 安直に `FontFamily` に複数の `Font` を指定しても、文字単位のフォールバックは実現できない
- Android 10（API 29）で追加された `Typeface.CustomFallbackBuilder` により、文字単位のフォールバックが実現できる
- `Typeface` を `AndroidFont` に変換することで、フォールバックの機能をComposeでも利用できる

## 参考文献

- [Typeface.CustomFallbackBuilder | Android Developers](https://developer.android.com/reference/kotlin/android/graphics/Typeface.CustomFallbackBuilder)
- [AndroidFon | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/ui/text/font/AndroidFont)
- [Google Fonts](https://fonts.google.com/)
