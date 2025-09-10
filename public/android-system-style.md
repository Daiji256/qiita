---
title: ユーザーの選んだスタイルを尊重する：Android アプリでのシステムスタイル取得
tags:
  - Android
  - Compose
  - UI
  - UX
  - MaterialDesign
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Android には、ユーザーが自分の好みや特性に合わせて設定できるシステムスタイルがあります。テーマ（ダークモード）、フォントスケール（フォントサイズ）、Material You（システムカラー）、色のコントラストなどが代表例です。これらはユーザーの「見やすさ」や「心地よさ」に直結する要素であり、ユーザーが意図的に設定した可能性が高いです。そのため、アプリがそれらの設定に沿うことで、ユーザー体験の向上につながります。

Material Design を全面的に採用すれば、これらのスタイルは自動的に反映されます。しかし、アプリ固有の世界観やデザインの自由度を優先するために、部分的もしくは全体で Material Design を採用しないという選択をとるでしょう。その場合でも「ユーザーの選んだスタイルを尊重する」ことは欠かせません。

本記事では、Android が提供する API を用いてシステムスタイルを取得する方法をまとめます。

## 背景

Android のスタイル設定は OS の進化とともに広がってきました。

テーマの切り替えやフォントサイズの変更から始まり、Material You によるシステムカラー、アクセシビリティを意識したコントラスト設定と発展しています。

これらは単なる好みのための調整だけではなく、ユーザーや環境に合わせた体験を維持すつための機能です。個々のアプリがそれらを無視すれば「読みにくい」、「見づらい」と感じられてしまいます。逆に適切に取り入れれば、Material Design を全面的に採用せずとも「ユーザーに配慮した UI」を実現できます。

## システムスタイルの取得方法

代表的なシステムスタイルの取得方法を紹介します。いずれも Compose の利用を前提としていますが、Compose を使わずとも実現できます。

### テーマ（ダークモード）

テーマは「システムがダークテーマであるか」として取得できます。テーマはユーザー体験に強く影響するため、簡単に取得できるように設計されています。

テーマは `isSystemInDarkTheme()` の戻り値が `false` ならライトテーマ、`true` ならダークテーマとして取得できます：

```kotlin
when (isSystemInDarkTheme()) {
    false -> { /* Light */ }
    true -> { /* Dark */ }
}
```

### フォントスケール

ユーザーはシステム設定でフォントサイズ（フォントスケール）を変更できます。フォントスケールは sp 単位に直接影響しているため、無意識的に実装した場合でもアプリにフォントスケールは反映されます。

フォントスケールの取りうる値は 0.85 – 2.00 倍の範囲です。すべてのフォントを 2 倍まで拡大すると大きくレイアウトが崩れる場合もあります。そのため、フォントスケールの値を参考に、レイアウトやタイポグラフィを調整する戦略も考えられます。

フォントスケールは `Density` から簡単に取得できます：

```kotlin
val density = LocalDensity.current
density.fontScale // 0.85f – 2.00f
```

### システムカラー（Material You, Android 12 / API 31 以降）

Android 12 で登場した Material You により、Android はシステムカラーを保持します。壁紙やユーザーの意図的な設定により、システムカラーが生成されます。

Material Design 3 のためのものですが、Material Design のライブラリを使用せずともこれらの値を取得可能です。取得したカラーパレットを独自の解釈で利用することもできます。また、その色から色相（Hue）を取得すれば、独自のパレットを生成することも可能でしょう。

システムカラーはカラーリソースに定義されているため、プライマリ・セカンダリ・ターシャリカラー全てにおいて簡単に取得できます。

この実装例では、プライマリカラーをテーマやコントラストを意識して取得しています：

```kotlin
@RequiresApi(Build.VERSION_CODES.S)
@Composable
fun primaryColor(): Color =
    when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE ->
            when (isSystemInDarkTheme()) {
                false -> colorResource(id = android.R.color.system_primary_light)
                true -> colorResource(id = android.R.color.system_primary_dark)
            }

        else ->
            when (isSystemInDarkTheme()) {
                false -> colorResource(id = android.R.color.system_accent1_400)
                true -> colorResource(id = android.R.color.system_accent1_800)
            }
    }
```

### コントラスト（Android 14 / API 34 以降）

Android 14 では、色のコントラスト設定が可能です。アクセシビリティを意識したもので前述のシステムカラーに影響をもたらします。

システムカラーを直接使わない場合においても、ユーザーが指定したコントラストに応じて配色を切り替えることは、良いユーザー体験につながります。

コントラストは `UiModeManager.contrast` から `Float` 値として取得できます。調査した限りでは、Default は 0、Medium は 0.5、High は 1 の値をとりました。

この実装例では、取得したコントラストを Default/Medium/High に分類しています：

```kotlin
@RequiresApi(Build.VERSION_CODES.UPSIDE_DOWN_CAKE)
@Composable
fun currentColorContrast(): ColorContrast {
    val context = LocalContext.current
    val uiModeManager = context.getSystemService(Context.UI_MODE_SERVICE) as UiModeManager
    val contrast = uiModeManager.contrast
    return when {
        contrast <= 1.0f / 3.0f -> ColorContrast.Default
        contrast <= 2.0f / 3.0f -> ColorContrast.Medium
        else -> ColorContrast.High
    }
}

enum class ColorContrast { Default, Medium, High }
```

## まとめ

Android にはテーマ、フォントスケール、システムカラー、コントラストなどユーザーが設定できるシステムスタイルが存在します。

これらを適切に利用することで、ユーザーの意思を反映しながらアプリ独自の世界観を維持できるようになります。Material Design を利用するかに関わらず、Android の進化に沿ったスタイル対応は、これからの Android アプリ開発において重要な取り組みです。
