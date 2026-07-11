---
title: newArticle001
tags:
  - Android
  - Compose
  - Roborazzi
  - Robolectric
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# ComposeとRoborazziでGoogle Play Store用の画像を生成してみた

## はじめに

個人でAndroidアプリを開発したとき、Google Play Store用のアイコンやスクリーンショットなどの画像アセットを用意するのが面倒だなと思いました。

アプリのUIやデザインを変更するたびに、Figmaなども更新し、画像をエクスポートするのは正直手間でした。個人的には、自分だけの開発なので、慣れ親しんだComposeやGit環境で完結する方が望ましいです。

そこで、普段UIテストで利用しているRoborazziを画像生成ツールとして流用してみました。RoborazziでGoogle Play Store用の画像を生成したことをこの記事でまとめます。

## 実装コード

使い慣れたComposeで画像用に描画し、Roborazziにより画像として出力しました。Robolectricの `@Config` により、画像サイズや言語設定を簡単に設定できます。

以下のようなUnitテストを作成し、実行するだけで指定したディレクトリに画像が生成されるようにしました。

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class CapturePlayStoreImages {
    @get:Rule
    val composeTestRule = createAndroidComposeRule<ComponentActivity>()

    // 512×512のアイコン画像を生成する（128×4=512）
    @Config(qualifiers = "w128dp-h128dp-xxxhdpi")
    @Test
    fun captureIcon() {
        composeTestRule.setContent {
            // ここに画像で出力したいComposableの実装を配置する
        }
        composeTestRule.onRoot().captureRoboImage(
            filePath = "play-store/icon.png",
        )
    }

    // スクリーンショットを生成
    @Config(qualifiers = "w414dp-h736dp-xxhdpi")
    @Test
    fun capturePhone_1() {
        // ...
    }

    // 言語をjaに設定して、スクリーンショットを生成
    @Config(qualifiers = "ja-w414dp-h736dp-xxhdpi")
    @Test
    fun capturePhone_1_ja() {
        // ...
    }
}
```

## 良かったこと

- AndroidのプロジェクトでGoogle Play Store用の画像を管理できるようになり、Figmaなどの外部ツールへの依存がなくなった
- UI変更などにより画像を差し替える必要が出てきた時、気が付きやすい

## まとめ

- TODO: 箇条書きで3つ程度

## 参考文献

- [Roborazzi](https://takahirom.github.io/roborazzi/top.html)
- [Robolectric](https://robolectric.org/)
