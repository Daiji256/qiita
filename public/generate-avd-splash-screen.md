---
title: ComposeとUnitテストでAnimatedVectorDrawableを作る
tags:
  - Android
  - robolectric
  - Compose
  - VectorDrawable
  - AnimatedVectorDrawable
private: false
updated_at: '2026-07-11T22:25:44+09:00'
id: 1dd5d3f28a834af793fd
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

Androidアプリを開発していると、ロゴ等のアニメーションを実現する方法の1つに `AnimatedVectorDrawable` があります。`AnimatedVectorDrawable` は公式にサポートされたアニメーションで、Android 12以降のスプラッシュ画面などでも利用されています。

一般的には、既存のアニメーション作成ツールやデザインツールから `AnimatedVectorDrawable` として出力することが多いでしょう。

しかし、本記事では「ツール探したり使い方覚えるのめんどくさいし、Kotlinのコードで作れた方がラクだな」と考え、Unitテストで `AnimatedVectorDrawable` を生成した話を紹介します。

なお、本記事では `AnimatedVectorDrawable` の詳しい仕様や、アニメーションの細かな調整方法については深く触れません。

## `AnimatedVectorDrawable` とは

`AnimatedVectorDrawable` とは、静的なベクター画像である `VectorDrawable` にアニメーション機能を搭載したものです。基本的には、同じような構造を持つベクター画像をいくつか用意し、その間の形状の変化（モーフィング）や、回転・スケールなどの動きを定義する形をとります。

本記事で紹介するアプローチでは、XMLの `<objectAnimator>` タグの中に、開始時のパス（`valueFrom`）から終了時のパス（`valueTo`）のセットをコマ送りのようにいくつか用意し、それらを順番に繋げることで滑らかなアニメーションを作成します。

## Composeの `Path` を使って `VectorDrawable` のためのパスを生成

Composeの `Path` には、SVGのパス文字列に変換するための `toSvg()` というAPIが用意されています。

`VectorDrawable` で使われる `pathData` はSVGのパスと同等です。そのため、Composeの `Path` をこねくり回して `toSvg()` を呼べば、そのまま `VectorDrawable` に使えるパス文字列が生成できます。つまり、Composeの `Path` の扱いに慣れている人なら、コードだけで自由にベクター画像を作れるといえます。

Composeの `Path` を使ったアニメーションを実現できているのであれば、それを時間経過のコマごとに細かく取り出すことで、`AnimatedVectorDrawable` に必要な `pathData` を簡単に集めることができます。

## Unitテストで `AnimatedVectorDrawable` を生成

なぜわざわざUnitテストを使うのかというと、単純に環境構築や実行が容易で、Androidエンジニアにとって慣れた実行環境だからです。`Robolectric` を併用することで、Androidのグラフィックス環境をシミュレートしながらCompose環境でのSVG出力までを実行できます。

以下のテストコードを実行することで、`res/drawable` に `animation.xml` を書き出すことができます。このコードでは、`Path` の作成に `androidx.graphics:graphics-shapes` の `Morph` を利用しています。

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class GenerateAnimatedVectorDrawable {
    @Test
    fun generate() {
        val xml = buildString {
            val morph = Morph(start = MaterialShapes.PixelCircle, end = MaterialShapes.Ghostish)
            val duration = 1000
            val n = 10
            fun svg(progress: Float): String =
                morph.toPath(progress = progress).apply {
                    translate(offset = Offset(0.5f, 0.5f) - this.getBounds().center)
                }.toSvg()

            appendLine("""<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"""")
            appendLine("""    xmlns:aapt="http://schemas.android.com/aapt">""")

            appendLine("""    <aapt:attr name="android:drawable">""")
            appendLine("""        <vector""")
            appendLine("""            android:height="108dp"""")
            appendLine("""            android:width="108dp"""")
            appendLine("""            android:viewportHeight="1"""")
            appendLine("""            android:viewportWidth="1">""")
            appendLine("""            <group>""")
            appendLine("""                <path""")
            appendLine("""                    android:name="path"""")
            appendLine("""                    android:fillColor="#000000"""")
            appendLine("""                    android:pathData="${svg(progress = 0f)}" />""")
            appendLine("""            </group>""")
            appendLine("""        </vector>""")
            appendLine("""    </aapt:attr>""")

            appendLine("""    <target android:name="path">""")
            appendLine("""        <aapt:attr name="android:animation">""")
            appendLine("""            <set android:ordering="sequentially">""")
            repeat(n) {
                appendLine("""                <objectAnimator""")
                appendLine("""                    android:duration="${duration / n}"""")
                appendLine("""                    android:propertyName="pathData"""")
                appendLine("""                    android:valueFrom="${svg(progress = it.toFloat() / n)}"""")
                appendLine("""                    android:valueTo="${svg(progress = (it.toFloat() + 1f) / n)}"""")
                appendLine("""                    android:valueType="pathType" />""")
            }
            appendLine("""            </set>""")
            appendLine("""        </aapt:attr>""")
            appendLine("""    </target>""")

            appendLine("""</animated-vector>""")
        }

        File("src/main/res/drawable/animation.xml").writeText(xml)
    }
}
```

## サンプルコードで得られる `AnimatedVectorDrawable` のプレビュー

前節のUnitテストを実行して得られた `animation.xml` は以下プレビューのようにアニメーションします。

![サンプルコードで得られるAnimatedVectorDrawableのプレビュー](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/8519aee0-8b09-4940-91f5-2ecf13110fa1.gif)

## まとめ

- デザインツールを使わず、使い慣れたKotlinとComposeにより `AnimatedVectorDrawable` を作成した
- Unitテストなら、Robolectricも利用できるし、慣れたやり方でファイルを作成できる

## 参考文献

- [AnimatedVectorDrawable | API reference | Android Developers](https://developer.android.com/reference/android/graphics/drawable/AnimatedVectorDrawable)
- [SplashScreen | API reference | Android Developers](https://developer.android.com/reference/android/window/SplashScreen)
- [Path | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/ui/graphics/Path)
