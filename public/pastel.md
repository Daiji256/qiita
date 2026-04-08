---
title: newArticle001
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# Material Youの動的配色を再定義する HCT色空間を用いた独自デザインの実装

AndroidのMaterial Design 3において導入されたMaterial You（Dynamic Color）は、ユーザーの壁紙から抽出された色を基にシステム全体に統一感をもたらします。しかし、標準の生成ロジックに依存するだけでは、アプリ固有の世界観やブランドカラーを表現しきれない場合があります。

本記事では、ユーザーのパーソナライズを尊重しつつ、パステルカラー調などの独自デザインを実現する手法について解説します。具体的には、Material Design 3の基盤であるHCT色空間を直接操作し、`androidx.core.graphics.ColorUtils.M3HCTToColor` を活用した色生成の実装方法を紹介します。

## Material Design 3のカラーシステムとHCT色空間

Material Design 3のカラーシステムは、従来のRGBやHSVではなく、HCTという新しい色空間に基づいています。HCTは、色相（Hue）、彩度（Chroma）、トーン（Tone）の3つの要素で構成されています。

この色空間は、人間の視覚特性を数学的にモデル化したCAM16と、知覚的な等色性を備えたLab色空間の利点を組み合わせて定義されました。最大の特徴は、トーンの値が視覚的な明るさ（Luminance）と直結している点にあります。これにより、異なる色相であってもトーンの値が同一であれば、人間には同じ明るさとして知覚されます。この特性は、アクセシビリティにおけるコントラスト比の計算を劇的に簡略化させました。



## ColorUtilsによる色空間の相互変換

AndroidのCore Ktxライブラリに含まれる `ColorUtils` を利用することで、HCTの値から色の生成や、既存の色からのHCT値の抽出が可能になります。Jetpack Composeでこれらを効率的に扱うために、以下のような拡張関数を定義して利用するのが一般的です。

`M3HCTToColor` は指定したHCT値を基にColorIntを返し、`colorToM3HCT` は既存の色をHCTの各要素に分解します。

```kotlin
@Stable
fun Color.Companion.m3Hct(hue: Float, chroma: Float, tone: Float): Color =
    Color(ColorUtils.M3HCTToColor(hue, chroma, tone))

@Stable
fun Color.m3Hue(): Float {
    val m3hct = FloatArray(3)
    ColorUtils.colorToM3HCT(this.toArgb(), m3hct)
    return m3hct[0]
}
```

## Material Youをベースとしたパステルカラー調の実装

ユーザーが選択したアクセントカラーの色相（Hue）を維持しつつ、彩度とトーンを固定または調整することで、独自のパレットを作成できます。例えば、アプリ全体を柔らかい雰囲気にしたい場合、彩度を一定以下に抑え、トーンを高めに設定することでパステルカラー調のカラースキームを動的に生成できます。

この手法を用いれば、ユーザーのパーソナライゼーションを損なうことなく、グラデーションの多用や特定のトーン＆マナーに合わせた独自の世界観を構築することが可能になります。

```kotlin
@Stable
fun dynamicLightPastelColorScheme(context: Context): ColorScheme {
    val mdColorScheme = dynamicLightColorScheme(context = context)
    val primaryHue = mdColorScheme.primary.m3Hue()
    val secondaryHue = mdColorScheme.secondary.m3Hue()

    return ColorScheme(
        primary = Color.m3Hct(hue = primaryHue, chroma = 50f, tone = 70f),
        secondary = Color.m3Hct(hue = secondaryHue, chroma = 30f, tone = 80f),
        surface = Color.m3Hct(hue = primaryHue, chroma = 8f, tone = 15f),
        onPrimary = Color.m3Hct(hue = primaryHue, chroma = 10f, tone = 100f),
        onSecondary = Color.m3Hct(hue = secondaryHue, chroma = 10f, tone = 100f),
        onSurface = Color.m3Hct(hue = primaryHue, chroma = 10f, tone = 90f),
        // その他の色も同様にHCTで定義
    )
}
```

## 実装における留意点

HCT色空間を利用した色の定義は強力ですが、アクセシビリティの観点からは注意が必要です。特にパステルカラーのようにトーンを高く設定する場合、背景色と文字色の間のコントラスト比が不足するリスクがあります。

本稿の実装例では簡略化のためにコントラストの自動調整は考慮していませんが、製品レベルの実装においては、Material Design 3のコントラストガイドラインに従い、十分な視認性が確保されているかを検証する必要があります。また、Material Design 3の仕様やドキュメントは活発に更新されており、AndroidのSDKバージョンによって生成されるパレットの挙動が異なる場合があることにも留意してください。

## 参考文献

* Material Design 3 - Color system overview: [https://m3.material.io/styles/color/system/overview](https://m3.material.io/styles/color/system/overview)
* Material Design 3 - The color system: [https://m3.material.io/styles/color/the-color-system/key-colors](https://m3.material.io/styles/color/the-color-system/key-colors)
* Androidアプリでのシステムスタイル取得 - Qiita: [https://qiita.com/Daiji256/items/52dafa49bec015700f43](https://qiita.com/Daiji256/items/52dafa49bec015700f43)
* androidx.core.graphics.ColorUtils Reference: [https://developer.android.com/reference/androidx/core/graphics/ColorUtils](https://developer.android.com/reference/androidx/core/graphics/ColorUtils)
