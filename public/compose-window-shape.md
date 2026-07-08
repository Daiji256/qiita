---
title: 端末の形状を頻繁に参照したくなったので、compositionLocalWithComputedDefaultOfを使って定義した
tags:
  - Android
  - Kotlin
  - Compose
  - JetpackCompose
  - CompositionLocal
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

スマホは画面の四隅が丸くなっています。一方で、`Composable` はデフォルトでは四角いです。画面遷移アニメーションを作っている時にその形状の差を埋めたいと考えました。これを実現するには、各画面の `Composable` も端末に合わせて形状を変更する必要があります。

![角丸を適用した時の画面遷移アニメーションの例](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/98cc46c8-9a9f-4aa0-bdf1-8a926b253ebc.png)

## アプローチ

角丸の半径は `WindowInsets.getRoundedCorner(position: Int)`[^android12] により取得できます。今回は、`LayoutDirection` を考慮した上で取得した半径を使って `Shape` を作成します。

この形状をさまざまな場所から参照するため、ここでは `compositionLocalWithComputedDefaultOf` を使って実装します。`compositionLocalWithComputedDefaultOf` を使うことで、デフォルト値に `LocalView` などを参照できます。

[^android12]: `WindowInsets.getRoundedCorner(position: Int)` はAndroid 12（API 31）以降に利用可能です。

## 実装コード

```kotlin
Scaffold(
    modifier = Modifier
        .clip(shape = LocalWindowShape.current),
) {
    // ...
}
```

```kotlin
val LocalWindowShape = compositionLocalWithComputedDefaultOf<Shape> {
    val view = LocalView.currentValue
    val insets = view.rootWindowInsets
    val topLeftCorner = insets?.getRoundedCorner(RoundedCorner.POSITION_TOP_LEFT)
    val topRightCorner = insets?.getRoundedCorner(RoundedCorner.POSITION_TOP_RIGHT)
    val bottomRightCorner = insets?.getRoundedCorner(RoundedCorner.POSITION_BOTTOM_RIGHT)
    val bottomLeftCorner = insets?.getRoundedCorner(RoundedCorner.POSITION_BOTTOM_LEFT)
    AbsoluteRoundedCornerShape(
        topLeft = topLeftCorner?.radius?.toFloat() ?: 0f,
        topRight = topRightCorner?.radius?.toFloat() ?: 0f,
        bottomRight = bottomRightCorner?.radius?.toFloat() ?: 0f,
        bottomLeft = bottomLeftCorner?.radius?.toFloat() ?: 0f,
    )
}
```

## 参考文献

- [Insets: Apply rounded corners | Views | Android Developers](https://developer.android.com/develop/ui/views/layout/insets/rounded-corners)
- [WindowInsets | API reference | Android Developers](https://developer.android.com/reference/android/view/WindowInsets)
- [LocalWindowShape.kt | Daiji256/android-showcase | GitHub](https://github.com/Daiji256/android-showcase/blob/e03373ad2062adda53c10393bca16593b335fe35/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/window/LocalWindowShape.kt)
