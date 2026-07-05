---
title: newArticle001
tags:
  - Gradle
  - Kotlin
  - Android
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# POM解析からJSON出力までを行うカスタムGradleプラグイン

ライブラリの依存関係やOSSライセンス情報を抽出する際、既存の多機能なツールでは「ブラックボックスな処理が多い」「不要な処理まで巻き込まれる」といった課題があります。

本記事では、依存関係のPOMを参照し、JSONファイルへの出力するための実装を紹介します。なお、記事中にはソースコード全体を載せることはありません。コード全体を見たい場合は[DependenciesPlugin.kt](https://github.com/Daiji256/android-showcase/blob/d17755437bbbcd90215d876354f29cc8133b21cb/build-logic/src/main/kotlin/io/github/daiji256/showcase/buildlogic/DependenciesPlugin.kt)を参照してください。

## コンセプト

- ランタイムでの依存は考えない（関心外）
- POMを情報源とし、それ以外は参照しない
- 独自のカスタマイズが可能

## POMの抽出と解析

このプラグインの中核は、JARファイル等ではなく **POMファイルそのものを直接取得して解析する** 処理です。

### `ArtifactResolutionQuery` によりPOMを参照

以下のコードはリリースビルドでランタイムに依存するモジュールのPOMを参照しています。

プロジェクトのコンフィグレーションから依存するモジュールの一覧を取得し、`ArtifactResolutionQuery` によりPOMの参照しています。

```kotlin
val moduleIds = project.configurations
    .filter { cfg ->
        cfg.isCanBeResolved && cfg.name == "releaseRuntimeClasspath"
    }
    .flatMap { cfg ->
        cfg.incoming.resolutionResult.allComponents
            .mapNotNull { it.id as? ModuleComponentIdentifier }
    }
val pomResults = project.dependencies
    .createArtifactResolutionQuery()
    .forComponents(moduleIds)
    .withArtifacts(MavenModule::class.java, MavenPomArtifact::class.java)
    .execute()
val pomFiles = pomResults.resolvedComponents.map { resolvedComponent ->
    resolvedComponent
        .getArtifacts(MavenPomArtifact::class.java)
        .filterIsInstance<ResolvedArtifactResult>()
        .single()
        .file
}
```

### POMを読む

取得したPOMファイルを `DocumentBuilderFactory` によって解析します。以下のコードでは、IDやライセンスなどを参照しています。

```kotlin
val pomFile: File = TODO()
val factory = DocumentBuilderFactory.newInstance().apply {
    setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)
}
val documentBuilder = factory.newDocumentBuilder()
val root = documentBuilder.parse(pomFile).documentElement

val groupId = root.text("groupId")
    ?: root.children("parent").firstOrNull()?.text("groupId")
    ?: error("missing groupId: ${pomFile.absolutePath}")
val artifactId = root.text("artifactId")
    ?: error("missing artifactId: ${pomFile.absolutePath}")
val id = "$groupId:$artifactId"
val name = root.text("name")
val url = root.text("url")
val license = root.children("licenses")
    .flatMap { it.children("license") }
    .firstOrNull()
    ?.text("name")
```

```kotlin
private fun Element.text(tagName: String): String? =
    children(tagName).firstOrNull()?.textContent?.ifEmpty { null }

private fun Element.children(tagName: String): List<Element> =
    buildList {
        for (index in 0 until childNodes.length) {
            val node = childNodes.item(index)
            if (node is Element && node.tagName == tagName) {
                add(node)
            }
        }
    }
```

## JSONの出力

### マニュアルでの上書き・追記

JSONに出力したい情報だとしても、POMに含まれていないケースや、POMとは値が異なるシーンがあります。それを上書きする設定を追加し、それが反映されるようにすると良いでしょう。

```json
{
  "dependencies": [
    {
      "id": "org.jspecify:jspecify",
      "url": "https://jspecify.dev/"
    },
    {
      "id": "material_symbols_and_icons",
      "name": "Material Symbols and Icons",
      "url": "https://fonts.google.com/icons",
      "license": "SIL Open Font License, Version 1.1"
    }
  ]
}
```

### JSONの出力

ここでは `kotlinx.serialization` により、JSONを作成してみます。`dependenciess` はPOMと任意の上書きによって得られたことを想定しています。

```kotlin
val dependenciess: List<Dependency> = TODO()
val json = Json.encodeToString(DependenciesMetadata(dependencies))
```

```kotlin
@Serializable
data class DependenciesMetadata(
    @SerialName("dependencies") val dependencies: List<Dependency>,
)

@Serializable
data class Dependency(
    @SerialName("id") val id: String,
    @SerialName("name") val name: String,
    @SerialName("url") val url: String,
    @SerialName("license") val license: String,
)
```

## **まとめ**

- TODO: 箇条書きで3つ程度
- 簡潔な文章で、「。」を付けなくて短いもの

## 参考文献

- [DependenciesPlugin.kt | Daiji256/android-showcase | GitHub](https://github.com/Daiji256/android-showcase/blob/d17755437bbbcd90215d876354f29cc8133b21cb/build-logic/src/main/kotlin/io/github/daiji256/showcase/buildlogic/DependenciesPlugin.kt)
