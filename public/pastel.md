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

Material YouによりGoogle製のアプリのみならず、サードパーティ製のアプリもユーザーの設定に応じたカラースキームを提供できます。一方で、自動で生成されたカラースキームをそのまま利用するだけでは、アプリ固有の世界観を表現しきれない場合があります[^material-3-expressive]。

[^material-3-expressive]: デザインの多様性のための1つの解決策として、Material 3 Expressiveがあります。

本記事では、ユーザーの設定を尊重しつつ独自のカラースキームを作成する方法について紹介します。

## Material Design 3のカラーシステムとHCT色空間

Material Design 3のカラーシステムは、よくあるRGBやHSLではなく、HCTという独自の色空間に基づいています。HCTは、色相（Hue）、彩度（Chroma）、トーン（Tone）の3つの要素で構成されています。

HCT色空間は、人間の知覚に基づいて設計されており、色の明るさや鮮やかさをより直感的に表現できるようになっています。HSL色空間などでは、色相を変更すると知覚的な明るさが変化してしまう問題がありましたが、HCTではトーンの値を固定することで、色相を変更しても視覚的な一貫性を保つことができます。これにより、ChromaやToneの値を調整することなく、Hueの変更のみでユーザーの設定に合わせたカラースキームを生成できます。

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

以下の実装例[^full-code][^contrast][^direct-resource]ではMaterial Design 3のprimaryカラー等から色相を抽出し、彩度とトーンを独自の値に設定することで、ユーザーの設定を反映しつつパステルカラーにしています。このように、HCT色空間で特定のパラメータを固定することで、ユーザーの設定とブランドアイデンティティを両立することが可能となります。

[^full-code]: 実装例は一部のみを抜粋しています。完全なコードは[Daiji256/android-showcase - PastelTheme.kt](https://github.com/Daiji256/android-showcase/blob/8094a2704549d9161b382731728132ae1f181799/feature/pastel/src/main/kotlin/io/github/daiji256/showcase/feature/pastel/PastelTheme.kt)にあります。

[^contrast]: Android 14に追加されたコントラスト設定を考慮した実装にはなっていません。

[^direct-resource]: `resources.getColor(android.R.color.system_accent1_500, theme)` のようにして、直接リソースから色を取得することもできます。。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/1fb397f9-79ef-4f4d-b22f-716961de6414.png" alt="パステルカラーのコンポーネントの例" width=320 />

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

## 注意点

自由に色を生成できる一方で、Material Design 3のようなアクセシビリティの確保は自分で担保する必要があります。また、ライト・ダークテーマやコントラスト設定などへの対応も、検討する必要があります。

また、Material Design 3の仕様は活発に更新されており、Androidのバージョンによって機能の変更・追加があります。

## まとめ

- Material Design 3のカラースキームとAndroidの仕組みを理解することで、独自の拡張・派生色を作成できる。
- `ColorUtils.M3HCTToColor` や `ColorUtils.colorToM3HCT` により、HCT色空間を扱うことができる。

## 参考文献

- [Color | Material Design 3](https://m3.material.io/styles/color/system/overview)
- [The science of color & design | Material Design 3](https://m3.material.io/blog/science-of-color-design)
- [ColorUtils | Android Developers](https://developer.android.com/reference/androidx/core/graphics/ColorUtils)
- [ユーザーの選んだスタイルを尊重する：Androidアプリでのシステムスタイル取得 | Qiita](https://qiita.com/Daiji256/items/52dafa49bec015700f43)
