---
title: OSSライセンスのためにPOMを参照してJSONを出力する
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

ライブラリの依存関係やOSSライセンス情報を抽出する際、既存の多機能なツールでは「ブラックボックスな処理が多い」「不要な処理まで巻き込まれる」といった課題があります。

本記事では、依存関係のPOMを参照しJSONを出力する実装を紹介します。なお、記事中にはソースコード全体を載せることはありません。コード全体を見たい場合は[DependenciesPlugin.kt](https://github.com/Daiji256/android-showcase/blob/d17755437bbbcd90215d876354f29cc8133b21cb/build-logic/src/main/kotlin/io/github/daiji256/showcase/buildlogic/DependenciesPlugin.kt)を参照してください。

## コンセプト

- ランタイムでは実行しない（事前にJSONを出力する想定）
- POMを情報源とし、それ以外は参照しない
- 独自のカスタマイズが可能

## POMの抽出と解析

### `ArtifactResolutionQuery` によりPOMを参照

以下のコードは、リリースビルドの実行時に必要な依存モジュール（`releaseRuntimeClasspath`）からPOMを参照しています。

プロジェクトのコンフィグレーションから依存するモジュールの一覧を取得し、`ArtifactResolutionQuery` によりPOMを参照しています。

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

### XMLパーサーによるPOMの解析と情報抽出

取得したPOMファイルを `DocumentBuilderFactory` によって解析します。

以下のコードでは、ID、バージョンやライセンス（複数ある場合は結合）などを抽出しています。

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
    .mapNotNull { it.text("name") }
    .joinToString(", ")
    .ifEmpty { null }
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

## データの補完とJSON出力

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

### Gradleタスクでの実行とJSONの出力

ここでは `kotlinx.serialization` によりJSONを作成します。`dependencies` はPOMと任意の上書きによって得られたことを想定しています。

```kotlin
val dependencies: List<Dependency> = TODO()
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

## まとめ

- 依存関係のグラフからPOMを直接取得
- セキュリティを考慮したXML解析と柔軟なカスタマイズ
- 足りない情報をマニュアルで補完しJSONとして出力

## 参考文献

- [POM Reference](https://maven.apache.org/pom.html)
- [DependenciesPlugin.kt | Daiji256/android-showcase | GitHub](https://github.com/Daiji256/android-showcase/blob/d17755437bbbcd90215d876354f29cc8133b21cb/build-logic/src/main/kotlin/io/github/daiji256/showcase/buildlogic/DependenciesPlugin.kt)
