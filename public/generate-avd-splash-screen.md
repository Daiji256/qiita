---
title: newArticle001
tags:
  - Android
  - Compose
  - VectorDrawable
  - AnimatedVectorDrawable
  - robolectric
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# Unitテストでスプラッシュ画面用のAnimatedVectorDrawableを作った話

Android 12以降はスプラッシュ画面の扱いには標準の `SplashScreen` APIを使います。スプラッシュ画面でアニメーションをするには、`AnimatedVectorDrawable` を使う必要があります。

一般的には、アニメーションを作成するツールから `AnimatedVectorDrawable` で出力することが多いでしょう。

本記事では「ツール探したり使い方覚えるのめんどくさいし、Kotlinのコードで作れた方がラクだな」と思った筆者による、Unitテストで `AnimatedVectorDrawable` を作ったことを紹介します。`AnimatedVectorDrawable` の詳しい仕様・技術、アニメーションの調整方法については触れません。

## `AnimatedVectorDrawable` とは

`AnimatedVectorDrawable` とは `VectorDrawable` にアニメーション機能を搭載したものです。基本的には、`VectorDrawable` と同様の仕組みのベクター画像をいくつか用意し、その間の動きを回転・スケールなどで補うという形をとっています。

次節以降で紹介するコードでは `<objectAnimator>` に「開始時のパス（`valueFrom`）」から「終了時のパス（`valueTo`）」のセットをいくつか用意し、それを繋げることでアニメーションを作成します。

### `AnimatedVectorDrawable` として必要なものを定義

TODO

```xml
<?xml version="1.0" encoding="utf-8"?>
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_splash_base">
    <target
        android:name="logo_path"
        android:animation="@animator/anim_splash" />
</animated-vector>
```

## Composeの `Path` を使って `VectorDrawable` のためのパスを生成

Composeの `Path` をSVGのパスに変換に変換するための `toSvg()` というAPIが用意されています。`VectorDrawable` のパスはSVGのパスと同等なため、`toSvg()` で `VectorDrawable` のパスを生成できると認識して良いでしょう。

つまり、Composeの `Path` の扱いに慣れているなら、`VectorDrawable` を作れるといって過言ではありません。

Composeの `Path` を使ってアニメーションを実現しているのであれば、それをコマごとに取り出し、`AnimatedVectorDrawable` のためのパスを集めることができます。

## Unitテストで `animator` を生成

`AnimatedVectorDrawable` はいくつかのファイルに分ける方法と、1つにまとめる方法があります。ここではいくつかのファイルに分ける方法を採用し、アニメーションの根幹部分である `animator` をUnitテストで生成することにします。

なぜUnitテストを使うかというと、環境構築や扱いが容易で、最も慣れている場だからです。`Robolectric` を使うことで、Compose環境でSVGの出力までを実行できます。

以下のテストコードを実行することで、`anim_splash.xml` を `res/animator` に書き出すことができました。コードから何をしているかある程度は伝わるかなと思います。

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class GenerateAnimSplash {
    @Test
    fun generate() {
        val xml = buildString {
            // ヘッダー
            appendLine(
                """
                    <?xml version="1.0" encoding="utf-8"?>
                    <set xmlns:android="http://schemas.android.com/apk/res/android"
                        android:ordering="sequentially">
                """.trimIndent(),
            )

            // パスを生成するためのMorph等
            val morph = Morph(start = MgShapes.FullCircle.polygon, end = MgShapes.M33.polygon)
            val duration = 400
            val n = 10
            fun progress(fraction: Float): Float =
                CubicBezierEasing(0.2f, 0f, 0f, 1f).transform(fraction = fraction)

            repeat(n) {
                // Morphから得られるPathをSVGに変換
                val valueFrom = morph.toPath(progress(fraction = it.toFloat() / n)).apply {
                    translate(offset = Offset(0.5f, 0.5f) - this.getBounds().center)
                }.toSvg()
                val valueTo = morph.toPath(progress(fraction = (it.toFloat() + 1f) / n)).apply {
                    translate(offset = Offset(0.5f, 0.5f) - this.getBounds().center)
                }.toSvg()

                // objectAnimatorを書き込む
                appendLine("""    <objectAnimator""")
                appendLine("""        android:duration="${duration / n}"""")
                appendLine("""        android:propertyName="pathData"""")
                if (it == 0) appendLine("""        android:startOffset="100"""")
                appendLine("""        android:valueFrom="$valueFrom"""")
                appendLine("""        android:valueTo="$valueTo"""")
                appendLine("""        android:valueType="pathType" />""")
            }

            // setを閉じる
            appendLine("""</set>""")
        }

        // res/animatorに書き出し
        File("src/main/res/animator/anim_splash.xml").writeText(xml)
    }
}
```

## まとめ

- TODO: 箇条書き3つ程度

## 参考文献

- [AnimatedVectorDrawable | API reference | Android Developers](developer.android.com/reference/android/graphics/drawable/AnimatedVectorDrawable)
- [SplashScreen | API reference | Android Developers](https://developer.android.com/reference/android/window/SplashScreen)
- [title>Path | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/ui/graphics/Path)
