---
title: Material Youを活かした独自カラースキームの作り方
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

Material Youは、Android 12で導入されたパーソナライズ機能であり、壁紙やユーザー設定に基づいてシステム全体のカラースキームを動的に生成します[^dynamic-color]。

[^dynamic-color]: Material Youによる色の動的生成は、Dynamic colorと呼ばれることがあります。

この仕組みにより、アプリはユーザーの好みに合わせた配色を実現しやすくなりました。一方で、標準の `dynamicLightColorScheme` / `dynamicDarkColorScheme` で生成された色をそのまま適用するだけでは、アプリ固有の世界観（パステル調、グラデーションなど）を十分に表現できないことがあります[^material-3-expressive]。

[^material-3-expressive]: デザインの多様性に対する1つのアプローチとして、Material 3 Expressiveがあります。

本記事では、`ColorUtils.M3HCTToColor` / `ColorUtils.colorToM3HCT` を用いて、ユーザー設定を活かした独自のカラースキームを作成する方法を紹介します。詳細なテーマ設計そのものではなく、HCT色空間を利用して配色を制御するアプローチに焦点を当てます。

## Material Design 3のカラーシステムとHCT色空間

Material Design 3では、従来のRGBやHSLではなく、HCT色空間を用いてカラースキームを定義しています。HCTは、色相（Hue）、彩度（Chroma）、トーン（Tone）の3要素で構成される、人間の知覚に基づいた色空間です。

従来のHSL色空間では、色相によって知覚上の明るさも変化するため、同じ明度や彩度を指定しても、色によって見え方やコントラストがそろわないことがあります。一方、HCTは人間の知覚特性を考慮して設計されているため、トーンを一定に保つことで、色相や彩度を変えても視覚的な明るさの一貫性を保ちやすくなります。この特性により、ユーザーが選んだ色から色相を取り出しつつ、彩度やトーンをアプリ側で制御しやすくなります。

## `ColorUtils` による色空間の相互変換

`androidx.core.graphics` の `ColorUtils` を使うと、HCT色空間との相互変換を行えます。`M3HCTToColor` は指定したHCT値から `@ColorInt Int` を返し、`colorToM3HCT` は既存の色をHCTの各要素に分解します。

`androidx.compose.ui.graphics.Color` を扱う場合は、以下のような拡張関数を定義しておくと便利です。

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

## ユーザー設定を活かしたパステル調の実装例

ユーザー設定に基づいて生成された色から色相を抽出し、彩度とトーンを独自に定義することで、パステル調のカラースキームを実現できます。

以下の実装例[^full-code][^contrast][^direct-resource]では、システムから取得した `primary` や `secondary` の色から色相のみを取り出し、彩度とトーンは独自に設定しています。

このように、ユーザー設定由来の色をそのまま使うのではなく、色相だけを参照して新たに色を作成することで、システムとの一貫性を保ちながら、アプリ独自の配色を構成できます。

[^full-code]: 実装例は一部のみを抜粋しています。完全なコードは[Daiji256/android-showcase - PastelTheme.kt](https://github.com/Daiji256/android-showcase/blob/8094a2704549d9161b382731728132ae1f181799/feature/pastel/src/main/kotlin/io/github/daiji256/showcase/feature/pastel/PastelTheme.kt)にあります。

[^contrast]: Android 14で追加されたコントラスト設定は考慮していません。

[^direct-resource]: `resources.getColor(android.R.color.system_accent1_500, theme)` のように、直接リソースから色を取得することもできます。

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

Material Design 3では、背景色と前景色のコントラスト比を十分に確保することで、アクセシビリティに配慮しています。また、Android 14で追加されたコントラスト強調設定にも対応しています。

独自のカラースキームを作成する場合、標準の動的カラースキームと同等の性質を必ずしも維持する必要はありません。ただし、実際の利用を考えると、アクセシビリティやコントラスト設定への配慮は重要です。特に、文字色と背景色の組み合わせによっては可読性が大きく損なわれるため、見た目だけでなくコントラストの検証も行うのが望ましいです。

また、Material Design 3の仕様やAndroid OSの実装は継続的に更新されます。独自実装はその影響を受けやすいため、OSバージョン差分やライブラリ更新に応じて、定期的に挙動を確認する必要があります。

## まとめ

- `ColorUtils.M3HCTToColor` や `ColorUtils.colorToM3HCT` を使うと、HCT色空間との相互変換を行える
- Material Youが生成した色から色相を取り出し、彩度やトーンを独自に設定することで、独自のカラースキームを作成できる
- 自由な配色を実現できる一方で、アクセシビリティやシステム設定（コントラストなど）への配慮は別途必要になる

## 参考文献

- [Color | Material Design 3](https://m3.material.io/styles/color/system/overview)
- [The science of color & design | Material Design 3](https://m3.material.io/blog/science-of-color-design)
- [ColorUtils | Android Developers](https://developer.android.com/reference/androidx/core/graphics/ColorUtils)
- [ユーザーの選んだスタイルを尊重する：Androidアプリでのシステムスタイル取得 | Qiita](https://qiita.com/Daiji256/items/52dafa49bec015700f43)
- [Daiji256/android-showcase - PastelTheme.kt | GitHub](https://github.com/Daiji256/android-showcase/blob/8094a2704549d9161b382731728132ae1f181799/feature/pastel/src/main/kotlin/io/github/daiji256/showcase/feature/pastel/PastelTheme.kt)
