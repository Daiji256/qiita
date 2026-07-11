---
title: ComposeとRoborazziでGoogle Play ストア用の画像を生成してみた
tags:
  - Android
  - robolectric
  - Compose
  - Roborazzi
private: false
updated_at: '2026-07-12T00:09:50+09:00'
id: bd52be069de7e8a70bed
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

個人でAndroidアプリを開発していたとき、Google Play ストア用のアイコンやスクリーンショットなどの画像アセットを用意するのが面倒だと感じました。

アプリのUIやデザインを変更するたびに、Figmaなどの外部ツール側も更新して、画像をエクスポートし直すのは正直手間です。個人的には、自分だけの開発だからこそ、慣れ親しんだComposeのコードとGitの環境だけで完結させたいと考えました。

そこで、普段UIテストで利用している[Roborazzi](https://takahirom.github.io/roborazzi/top.html)を画像生成ツールとして流用してみました。やってみたら便利だったので、設定方法や良かったことをこの記事にまとめます。

## 実装コード

使い慣れたComposeで画像用にUIを描画し、Roborazziの機能を使ってローカルに画像を出力します。

Robolectricの `@Config(qualifiers = "...")` により画像のサイズや端末の言語設定をテストケースごとに設定できます。以下のようなUnitテストを作成し、実行するだけで、指定したディレクトリに画像を生成できます。

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class CapturePlayStoreImages {
    @get:Rule
    val composeTestRule = createAndroidComposeRule<ComponentActivity>()

    // Google Play ストア用アイコン（512px × 512px）を生成する（128dp × 4(xxxhdpi) = 512px）
    @Config(qualifiers = "w128dp-h128dp-xxxhdpi")
    @Test
    fun captureIcon() {
        composeTestRule.setContent {
            // ここに画像として出力したいComposableの実装を配置する
            AppIcon()
        }
        composeTestRule.onRoot().captureRoboImage(
            filePath = "play-store/icon.png",
        )
    }

    // 通常のスマートフォンサイズのスクリーンショットを生成
    @Config(qualifiers = "w414dp-h736dp-xxhdpi")
    @Test
    fun capturePhone_1() {
        // ...
    }

    // 言語を日本語（ja）に設定して、スクリーンショットを生成
    @Config(qualifiers = "ja-w414dp-h736dp-xxhdpi")
    @Test
    fun capturePhone_1_ja() {
        // ...
    }
}
```

## 良かったこと

- Androidプロジェクト内でGoogle Play ストア用の画像を管理・生成できるようになり、Figmaなどの外部ツールの利用が減る
- UIコンポーネントやカラー定義を変更した際、テストを再実行するだけで画像が更新されるため、ストア画像の更新忘れを防止できる

## まとめ

- Roborazziをテスト用ではなく、画像生成の用途で利用しても便利
- Robolectricの `@Config` を活用すれば、Roborazziで生成する画像の設定が可能
- Androidプロジェクト内でGoogle Play ストア用の画像を管理したい場合、Roborazziは有用

## 参考文献

- [Roborazzi](https://takahirom.github.io/roborazzi/top.html)
- [Robolectric](https://robolectric.org/)
