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

## 主要要素

- Material Youを活かしつつ独自デザインを作ること
- ユーザーの意図と世界観を両立できる
- `androidx.core.graphics.ColorUtils.M3HCTToColor` の紹介
- 具体例としてのパステルカラー調

## 内容

- Material Youを尊重しつつ、アプリ独自のデザインにしたい
  - 彩度やトーンを自由に決めたい
  - グラデーションのために色相やトーン等を調整したい
  - secondaryやtertiary以外にも派生の色を扱いたい
  - 用意されている色だけだと、似たり寄ったりな世界観になるので、独自に調整したい
  - ユーザーの意図（設定されたMaterial Youの色）は尊重したい
  - 例えば、パステルカラー調にして、やわら無い雰囲気にするなど
  - 他にも、グラデーションの多用やかっこいい系なども可能
- Material Design 3のカラーシステムについて
  - 色相（Hue）、彩度（Chroma）、トーン（Tone）による色空間を定義
  - HCT色空間は一般的な色空間ではなく、Material Design 3のために定義された色空間
  - HCT色空間はCAM16とLabの組み合わせで定義されている
  - Material Youでは、ユーザーが選択した色かカラーテーマを作成する
- パステルカラーの実装例
  - あくまでも実装例
  - chromaやtoneの値は、適当に決めた値（footnote）
  - 実装例では、コントラスト設定は考慮していない（footnote）
  - 十分なコントラストの確保などは、デザインの観点で考慮する必要あり（footnote）
- `androidx.core.graphics.ColorUtils.M3HCTToColor` の紹介
  - `ColorUtils.M3HCTToColor` により、HCT色空間を用いた色を定義できる
  - `ColorUtils.colorToM3HCT` を用いれば、HCT色空間の値に変換もできる
  - Composableで扱うには、ColorIntをColorに変換する必要がある
- 「Androidアプリでのシステムスタイル取得」の紹介
  - 色相以外にも、ライト・ダークテーマやコントラストなども考慮できる
  - 「Androidアプリでのシステムスタイル取得」が参考になる
- Material Design 3のカラーシステムは日々進化
  - 補足程度（footnoteかも）
  - Androidのバージョン（SDKバージョン）によっても異なる
  - Material Design 3のドキュメントも更新されている（それなりに活発）

## ソースコード

### HCTからColorへの変換

```kotlin
@Stable
fun Color.Companion.m3Hct(hue: Float, chroma: Float, tone: Float): Color =
    Color(ColorUtils.M3HCTToColor(hue, chroma, tone))
```

### ColorからHCTへの変換

```kotlin
@Stable
fun Color.m3Hue(): Float {
    val m3hct = FloatArray(3)
    ColorUtils.colorToM3HCT(this.toArgb(), m3hct)
    return m3hct[0]
}
```

### Material Youの色をベースに、独自のパステルカラー調のカラースキームを定義

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

## 添付できる画像

- Material Youによるカラーテーマと、パステルカラー調のカラーテーマの比較画像
- パステルカラー時のコンポーネントの表示例
