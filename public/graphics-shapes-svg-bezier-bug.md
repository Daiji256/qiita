---
title: 'androidx.graphics:graphics-shapesのSvgPathParserにおけるベジェ曲線変換バグと回避策'
tags:
  - Android
  - SVG
  - Kotlin
  - Jetpack
  - RoundedPolygon
private: false
updated_at: '2026-07-04T08:31:37+09:00'
id: 7d2e4a85f55c84b38c34
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

`androidx.graphics:graphics-shapes:1.1.0` にて追加された `SvgPathParser.parseFeatures()` を使用した際、SVGのパスが正しくパースされず、形状が歪むバグに遭遇しました。

本記事では、その原因とコード上で解決する方法を紹介します。

なお、本件はすでにGoogle Issue Trackerに報告済みです：[Issue Tracker #527243511](https://issuetracker.google.com/issues/527243511)。

## 事象・原因

`SvgPathParser.parseFeatures()` を使ってSVGパス文字列から `features` を取得し、それを `RoundedPolygon` に変換して描画したところ、本来の形状より歪んでしまう現象が発生します。

SVGパスに含まれる2次ベジェ曲線（`q`コマンドなど）を、ライブラリ内部で3次ベジェ曲線に変換して扱う際は、2/3の係数を用いた座標計算が必要です。しかし、[ソースコード](https://github.com/androidx/androidx/blob/e6d33dd5d0a60001a5784d84123b05308d35f410/graphics/graphics-shapes/src/commonMain/kotlin/androidx/graphics/shapes/SvgPathParser.kt)を確認したところ、2次ベジェ曲線の制御点がそのまま3次ベジェ曲線の2つの制御点として単純にコピーされていました：

```kotlin
'q' -> addCurveWith(command.xy(0, 1), command.xy(0, 1), command.xy(2, 3))
```

| パスが歪んでしまった形状の例 | 期待している形状の例 |
| -- | -- |
| ![actual_distorted_curves.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/a9317fcd-689a-4e04-a11b-c83209ed09eb.png) | ![expected_correct_curves.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/d6d70811-a146-4b8e-beaf-51d922afa814.png) |

## 解決方法

`SvgPathParser.parseFeatures()` は、リリースされるアプリの本番環境（実行時）で動的に実行することは推奨されておらず、あくまでも開発中やビルド時のみの利用が想定されています。そのため、ここでは取得した `Feature` に直接パッチを当てて補正することで解決をはかります。

パース結果として得られた `Feature` の中から、ベジェ曲線を保持する `Cubic` クラスをチェックします。2つの制御点が完全に同一座標にある場合、単純コピーされた2次ベジェ曲線と判断し、本来の2/3の係数がついた制御点になるよう再計算・上書きします：

```kotlin
val features = SvgPathParser.parseFeatures(
    svgPath = "...",
).map { feature ->
    val patchedCubics = feature.cubics.map { cubic ->
        // 2つの制御点が同一座標の場合、単純コピーされた2次ベジェ曲線とみなす
        if (cubic.control0X == cubic.control1X && cubic.control0Y == cubic.control1Y) {
            Cubic(
                anchor0X = cubic.anchor0X,
                anchor0Y = cubic.anchor0Y,
                anchor1X = cubic.anchor1X,
                anchor1Y = cubic.anchor1Y,
                // 本来必要な2/3の係数を適用して制御点を再計算
                control0X = cubic.anchor0X + (cubic.control0X - cubic.anchor0X) * 2f / 3f,
                control0Y = cubic.anchor0Y + (cubic.control0Y - cubic.anchor0Y) * 2f / 3f,
                control1X = cubic.anchor1X + (cubic.control0X - cubic.anchor1X) * 2f / 3f,
                control1Y = cubic.anchor1Y + (cubic.control0Y - cubic.anchor1Y) * 2f / 3f,
            )
        } else {
            cubic
        }
    }

    when {
        feature.isIgnorableFeature -> Feature.buildIgnorableFeature(cubics = patchedCubics)
        feature.isEdge -> Feature.buildEdge(cubic = patchedCubics.single())
        feature.isConvexCorner -> Feature.buildConvexCorner(cubics = patchedCubics)
        feature.isConcaveCorner -> Feature.buildConcaveCorner(cubics = patchedCubics)
        else -> error("unknown feature type")
    }
}
```

## 参考文献

- [SvgPathParser incorrectly converts Quadratic Bezier (missing 2/3 factor) | Google Issue Tracker](https://issuetracker.google.com/issues/527243511)
- [androidx.graphics.shapes.SvgPathParser | GitHub](https://github.com/androidx/androidx/blob/e6d33dd5d0a60001a5784d84123b05308d35f410/graphics/graphics-shapes/src/commonMain/kotlin/androidx/graphics/shapes/SvgPathParser.kt)
- [SvgPathParser | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/graphics/shapes/SvgPathParser)
- [RoundedPolygon | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/graphics/shapes/RoundedPolygon)
- [Material Symbols and Icons | Google Fonts](https://fonts.google.com/icons)
