---
title: Nav3用の汎用的なpredictivePopTransitionSpecを実装した
tags:
  - Android
  - Compose
  - JetpackCompose
  - PredictiveBack
  - Navigation3
private: false
updated_at: '2026-07-08T22:41:14+09:00'
id: ee591d57616dd1504526
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

Navigation 3（Nav3）の `NavDisplay` では、通常の戻るアニメーション（`popTransitionSpec`）と、予測型「戻る」（Predictive back）アニメーション（`predictivePopTransitionSpec`）を別々に設定できるようになりました。

本記事では、それっぽい汎用的な `predictivePopTransitionSpec` を実装したので共有します。

## 意図と実装

Googleの意図としては「状況に適したアニメーションを設定してほしい」ということだと思われます。しかし、すべての画面遷移で個別に考えるのは現実的ではないため、まずはアプリ全体で使える「標準的なもの」を1つ用意するのが良いと考えました。

ここで紹介するのは、`Activity` の予測型「戻る」に適用される「あのデフォルト画面遷移」っぽい雰囲気になる実装です。完全に一致するわけではありませんが、特別なアニメーションを設定しない通常の画面遷移の場合に役立つと思います。

![予測型「戻る」の動き](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/a7429428-eb22-438a-8781-4c574c8c5e25.gif)

```kotlin
val predictivePopTransitionSpec: AnimatedContentTransitionScope<*>.(
    swipeEdge: @NavigationEvent.SwipeEdge Int,
) -> ContentTransform = { swipeEdge ->
    ContentTransform(
        targetContentEnter = scaleIn(
            initialScale = 0.875f,
            transformOrigin = TransformOrigin(
                pivotFractionX = -0.5f,
                pivotFractionY = 0.5f,
            ),
            animationSpec = tween(
                durationMillis = PredictivePopTransitionDelayedDuration,
                delayMillis = PredictivePopTransitionDelay,
                easing = StandardDecelerateEasing,
            ),
        ),
        initialContentExit = scaleOut(
            targetScale = 0.75f,
            transformOrigin = TransformOrigin(
                pivotFractionX = when (swipeEdge) {
                    NavigationEvent.EDGE_LEFT -> 0.625f
                    NavigationEvent.EDGE_RIGHT -> 0.375f
                    else -> 0.5f
                },
                pivotFractionY = 0.5f,
            ),
            animationSpec = tween(
                durationMillis = TransitionDuration,
                easing = StandardAccelerateEasing,
            ),
        ) +
            fadeOut(
                animationSpec = tween(
                    durationMillis = PredictivePopTransitionDelayedDuration,
                    delayMillis = PredictivePopTransitionDelay,
                    easing = StandardAccelerateEasing,
                ),
            ),
    )
}

private const val TransitionDuration = 300
private const val PredictivePopTransitionDelay = TransitionDuration * 4 / 5
private const val PredictivePopTransitionDelayedDuration =
    TransitionDuration - PredictivePopTransitionDelay
private val StandardDecelerateEasing = CubicBezierEasing(0f, 0f, 0f, 1f)
private val StandardAccelerateEasing = CubicBezierEasing(0.3f, 0f, 1f, 1f)
```

## 参考文献

- [NavDisplay | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/navigation3/ui/NavDisplay.composable)
- [ShowcaseAnimations.kt | Daiji256/android-showcase | GitHub](https://github.com/Daiji256/android-showcase/blob/d2267fcb29979efba3ed77bf78ee101694cda6a6/core/designsystem/src/main/kotlin/io/github/daiji256/showcase/core/designsystem/theme/ShowcaseAnimations.kt)
