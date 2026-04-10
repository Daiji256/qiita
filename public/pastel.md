---
title: Material YouとHCT色空間を用いたカラースキームの再定義
tags:
  - Android
  - Compose
  - JetpackCompose
  - MaterialDesign
  - UI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Material Youは、Android 12から導入されたパーソナライズ機能であり、壁紙やユーザー設定に基づいてシステム全体のカラースキームを動的に生成します[^dynamic-color]。

[^dynamic-color]: Material Youによる色の生成は、Dynamic Colorsと呼ばれる場合があります。

この仕組みにより、アプリはユーザーの好みに即座に適応できるようになりました。しかし、標準の `dynamicLightColorScheme` / `dynamicDarkColorScheme` で生成された色をそのまま適用するだけでは、アプリ固有の世界観（パステル調、グラデーションなど）を表現しきれないという課題があります[^material-3-expressive]。

[^material-3-expressive]: デザインの多様性のための1つの解決策として、Material 3 Expressiveがあります。

本記事では、`ColorUtils.M3HCTToColor` / `ColorUtils.colorToM3HCT` を用いることで、ユーザー設定を活かした独自のカラースキームを構築できることを紹介します。詳細なテーマ設計ではなく、その可能性に焦点を当てます。

## Material Design 3のカラーシステムとHCT色空間

Material Design 3では、従来のRGBやHSLではなく、HCT色空間によりカラースキームを定義しています。HCTは、色相（Hue）、彩度（Chroma）、トーン（Tone）の3要素で構成される人間の知覚に基づいた色空間です。

従来のHSL色空間では、色相によって知覚的な明るさも変わるため、コントラスト比が崩れやすいです。一方、HCTは人間の知覚特性に基づいて設計されているため、トーンを固定にすることで、色相や彩度を変えても視覚的な明るさの一貫性を保つことができます。この特性により、ユーザーが選んだ色（色相）を抽出しつつ、彩度や明るさをアプリ側で制御できます。

## `ColorUtils` による色空間の相互変換

`androidx.core.graphics` の `ColorUtils` により、HCT色空間を扱えます。`M3HCTToColor` は指定したHCT値から `@ColorInt Int` を返し、`colorToM3HCT` は既存の色をHCTの各要素に分解します。

`androidx.compose.ui.graphics.Color` に対して扱う場合は、以下のような拡張関数を定義すると便利です。

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

## ユーザー設定を活かしたパステルカラー調の実装例

ユーザー設定に基づく色から色相を抽出し、彩度とトーンを独自に定義することで、パステルカラー調のカラースキームを実現できます。

以下の実装例[^full-code][^contrast][^direct-resource]では、システムから取得した `primary` や `secondary` の色相のみを継承し、彩度とトーンを独自に設定しています。

[^full-code]: 実装例は一部のみを抜粋しています。完全なコードは[Daiji256/android-showcase - PastelTheme.kt](https://github.com/Daiji256/android-showcase/blob/8094a2704549d9161b382731728132ae1f181799/feature/pastel/src/main/kotlin/io/github/daiji256/showcase/feature/pastel/PastelTheme.kt)にあります。

[^contrast]: Android 14に追加されたコントラスト設定を考慮した実装にはなっていません。

[^direct-resource]: `resources.getColor(android.R.color.system_accent1_500, theme)` のようにして、直接リソースから色を取得することもできます。

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

## 実装における留意事項

Material Design 3では、背景と前面のコントラスト比を十分に確保することで、アクセシビリティを担保しています。また、Android 14で追加されたコントラストの強調設定にも対応しています。

独自のカラースキームを生成するとき、それらを必ず担保する必要はありません。ただし、ユーザーのことを考えると、アクセシビリティやコントラスト設定は考慮したほうがよいです。

また、Material Design 3の仕様は更新され続けており、Android OSのバージョンによって内部的な変更が入ることもあります。独自実装はその影響を受けやすく、定期的なメンテナンスが必要になる可能性があります。

## まとめ

- `ColorUtils.M3HCTToColor` や `ColorUtils.colorToM3HCT` により、HCT色空間を扱える
- Material Youが生成した色を参考に、独自のカラースキームを構築できる
- 自由な色生成が可能な一方で、アクセシビリティやシステム設定（コントラスト等）は別途考慮が必要になる

## 参考文献

- [Color | Material Design 3](https://m3.material.io/styles/color/system/overview)
- [The science of color & design | Material Design 3](https://m3.material.io/blog/science-of-color-design)
- [ColorUtils | Android Developers](https://developer.android.com/reference/androidx/core/graphics/ColorUtils)
- [ユーザーの選んだスタイルを尊重する：Androidアプリでのシステムスタイル取得 | Qiita](https://qiita.com/Daiji256/items/52dafa49bec015700f43)
- [Daiji256/android-showcase - PastelTheme.kt | GitHub](https://github.com/Daiji256/android-showcase/blob/8094a2704549d9161b382731728132ae1f181799/feature/pastel/src/main/kotlin/io/github/daiji256/showcase/feature/pastel/PastelTheme.kt)
