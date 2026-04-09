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

# Material Youと独自の世界観を両立させる実装

## はじめに

Material Youは、Android 12から導入された自分好みの色合いやデザインに変える機能です。端末の壁紙やユーザーの設定に基づいて、システム全体のカラースキームが動的に生成されます[^dynamic-color]。

[^dynamic-color]: Material Youによる色の生成は、Dynamic Colorsと呼ばれる場合があります。

Material YouによりGoogle製のアプリのみならず、サードパーティ製のアプリもユーザーの設定に応じたカラースキームを提供できるます。一方で、自動で生成されたカラースキームをそのまま利用するだけでは、アプリ固有の世界観を表現しきれない場合があります[^material-3-expressive]。

[^material-3-expressive]: デザインの多様性のための1つの解決策として、Material 3 Expressiveがあります。

本記事では、ユーザーの設定を尊重しつつ独自のカラースキームを作成する方法について紹介します。

## Material Design 3のカラーシステムとHCT色空間

Material Design 3のカラーシステムは、よくあるRGBやHSLではなく、HCTという独自の色空間に基づいています。HCTは、色相（Hue）、彩度（Chroma）、トーン（Tone）の3つの要素で構成されています。

HCT色空間は、人間の知覚に基づいて設計されており、色の明るさや鮮やかさをより直感的に表現できるようになっています。これにより、chromaやtoneの値を調整することなく、hueの変更のみでユーザーの設定に合わせたカラースキームを生成できます。

## `ColorUtils` による色空間の相互変換

`androidx.core.graphics.ColorUtils` を利用することで、簡単にHCT色空間を扱うことができます。`M3HCTToColor` は指定したHCT値を基に `@ColorInt Int` を返し、`colorToM3HCT` は既存の色をHCTの各要素に分解します。

Composeでの利用を想定するなら、`ColorUtils` をラップする形で、HCT色空間を直接扱えるような拡張関数を定義すると便利です。

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

ユーザー設定を尊重した独自のカラースキームの作成の例として、パステルカラー調のカラースキームを生成する方法を紹介します。

Material Design 3のprimaryカラー等から色相を抽出し、彩度とトーンを独自の値に設定することで、ユーザーの選択を反映しつつパステルカラーにしています。

```kotlin
@Stable
fun dynamicLightPastelColorScheme(context: Context): ColorScheme {
    val mdColorScheme = dynamicLightColorScheme(context = context)
    val primaryHue = mdColorScheme.primary.m3Hue()
    val secondaryHue = mdColorScheme.secondary.m3Hue()
    // ...
    return ColorScheme(
        primary = Color.m3Hct(hue = primaryHue, chroma = 50f, tone = 70f),
        secondary = Color.m3Hct(hue = secondaryHue, chroma = 30f, tone = 80f),
        surface = Color.m3Hct(hue = primaryHue, chroma = 8f, tone = 15f),
        // ...
    )
}
```

| Material Design 3 | パステルカラー |
| --- | --- |
| ![Material Design 3の色一覧](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/5108c210-f9f5-4774-9ccf-f74a34bf1ec2.png) | ![パステルカラーの色一覧](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/effb3ddf-db7e-4908-b7e8-83ca8a2ae1f0.png) |
| ![Material Design 3によるコンポーネント表示例](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/0372d5ec-2280-4356-b238-08e0b91902ce.png) | ![パステルカラーによるコンポーネント表示例](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/1fb397f9-79ef-4f4d-b22f-716961de6414.png) |

## 注意点

独自のカラースキームを作成する際にはいくつかの注意点があります。

色相だけでなく、ライト・ダークテーマやコントラストの調整も考慮すると良いでしょう。

Material Design 3ではアクセシビリティの観点から、コントラスト比を十分に確保しています。一方で、独自実装の場合はその縛りはありません。アクセシビリティを考慮したデザインが好ましいです。

## まとめ

TODO: 箇条書き3点程度でまとめる

## 参考文献

- [Color | Material Design 3](https://m3.material.io/styles/color/system/overview)
- [The science of color & design | Material Design 3](https://m3.material.io/blog/science-of-color-design)
- [ColorUtils | Android Developers](https://developer.android.com/reference/androidx/core/graphics/ColorUtils)
- [ユーザーの選んだスタイルを尊重する：Androidアプリでのシステムスタイル取得 | Qiita](https://qiita.com/Daiji256/items/52dafa49bec015700f43)
