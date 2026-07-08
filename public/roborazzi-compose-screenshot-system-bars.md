---
title: newArticle001
tags:
  - Android
  - Compose
  - JetpackCompose
  - Roborazzi
  - テスト
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

- Roborazziを使ってPreviewをスクリーンショット撮り、差分を検出している
- デフォルトでは、システムバーを含めない
- システムバーをスクリーンショットの対象に加えることで、`navigationBarsPadding()` の実装ミスなどを見つけやすくなる
- スクリーンショットはテスト以外の目的（他職種への共有）などでも使いまわすため、自然な見た目でシステムバーを実装した
- `DeviceConfigurationOverride` + `DeviceConfigurationOverride.WindowInsets` で `WindowInset` を設定
- `Modifier.testTag("root")` と `.onNodeWithTag(testTag = "root")` で `DeviceConfigurationOverride` 下の `Composable` のスクリーンショットを撮る
- `SystemUi()` でシステムバーに相当するデザインを表示

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class SystemBarsTest {
    @get:Rule
    val composeTestRule = createAndroidComposeRule<ComponentActivity>()

    @Config(qualifiers = "w414dp-h736dp-xxhdpi")
    @Test
    fun test() {
        composeTestRule.setContent {
            DeviceConfigurationOverride(
                override = DeviceConfigurationOverride.WindowInsets(windowInsets = windowInsets()),
            ) {
                Box(modifier = Modifier.testTag("root")) {
                    // 適当にコメント
                    Box(
                        modifier = Modifier
                            .fillMaxSize()
                            .background(Color(0xFFE0E0E0))
                            .systemBarsPadding()
                            .background(Color(0xFFE8E8E8))
                    ) {
                        Text(text = "text")
                    }

                    SystemUi()
                }
            }
        }
        composeTestRule
            .onNodeWithTag(testTag = "root")
            .captureRoboImage(filePath = "image.png")
    }
}
```

```kotlin
@Composable
fun windowInsets() =
    with(LocalDensity.current) {
        WindowInsetsCompat.Builder()
            .setInsets(
                WindowInsetsCompat.Type.statusBars(),
                Insets.of(0, 24.dp.roundToPx(), 0, 0),
            )
            .setInsets(
                WindowInsetsCompat.Type.navigationBars(),
                Insets.of(0, 0, 0, 24.dp.roundToPx()),
            )
            .build()
    }
```

```kotlin
@Composable
fun SystemUi() {
    Column {
        Row(
            verticalAlignment = Alignment.Bottom,
            modifier = Modifier
                .fillMaxWidth()
                .height(24.dp)
                .padding(horizontal = 4.dp),
        ) {
            Text(
                text = "15:00",
                style = TextStyle(),
                fontSize = with(LocalDensity.current) { 16.dp.toSp() },
                color = Color.Gray,
            )
            Spacer(modifier = Modifier.weight(1f))
            Icon(
                painter = painterResource(id = R.drawable.ic_cell_bar),
                contentDescription = null,
                tint = Color.Gray,
            )
            Spacer(modifier = Modifier.width(4.dp))
            Icon(
                painter = painterResource(id = R.drawable.ic_wifi),
                contentDescription = null,
                tint = Color.Gray,
            )
            Spacer(modifier = Modifier.width(4.dp))
            Icon(
                painter = painterResource(id = R.drawable.ic_battery),
                contentDescription = null,
                tint = Color.Gray,
            )
        }
        Spacer(modifier = Modifier.weight(1f))
        Box(
            contentAlignment = Alignment.Center,
            modifier = Modifier
                .fillMaxWidth()
                .height(24.dp),
        ) {
            Box(
                modifier = Modifier
                    .size(width = 100.dp, height = 4.dp)
                    .background(color = Color.Gray, shape = CircleShape),
            )
        }
    }
}
```
