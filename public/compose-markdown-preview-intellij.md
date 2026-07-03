---
title: newArticle001
tags:
  - Kotlin
  - Compose
  - Markdown
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# intellij-markdownとComposeでMarkdownを柔軟に描画する

本記事では、ComposeでMarkdownを描画する方法について紹介します。既存のMarkdown描画ライブラリを使うのではなく、[intellij-markdown](https://github.com/JetBrains/markdown)によりMarkdownをパースし、Composeで描画するという手順を踏みます。

Markdownのパースを[intellij-markdown](https://github.com/JetBrains/markdown)に任せることで、多くの複雑な処理から解放されます。またパースして得られた構文木（AST）をどのように描画するかを自前で実装し独自のデザインや拡張性などの自由度を確保します。

## Markdownをパース

`MarkdownParser` によりMarkdown形式の `String` をパースし、`ASTNode` を得ます。

```kotlin
val markdown = "#Header\nParagraph."
val markdownParser = MarkdownParser(CommonMarkFlavourDescriptor())
val node: ASTNode = markdownParser.buildMarkdownTreeFromString(text = markdown)
```

## `ASTNode` を扱いやすい形に変換

`ASTNode` はMarkdownの強調表示やコード表示などがネストされたものを表します。そのため、そのまま描画するには扱いが難しいです。

ここでは、`ASTNode` を描画で扱いやすい独自クラスを定義し変換します。この、`Block` と `Content` はComposeで描画することを意識し、`Composable` や `AnnotatedString` で扱いやすいです。

<details><summary>変換処理</summary>

```kotlin
fun ASTNode.toBlocks(markdown: String): List<Block> =
    when (this.type) {
        MarkdownTokenTypes.EOL ->
            emptyList()

        MarkdownElementTypes.MARKDOWN_FILE ->
            children.flatMap { it.toBlocks(markdown) }

        MarkdownElementTypes.PARAGRAPH ->
            listOf(
                Block.Paragraph(
                    contents = children
                        .mapNotNull { it.toContent(markdown) },
                ),
            )

        MarkdownElementTypes.ATX_1 ->
            listOf(
                Block.H1(
                    contents = children
                        .drop(1) // drop #
                        .mapNotNull { it.toContent(markdown) },
                ),
            )

        MarkdownElementTypes.CODE_FENCE ->
            listOf(
                Block.Code(
                    contents = children
                        .filterNot { it.type == MarkdownTokenTypes.CODE_FENCE_START }
                        .filterNot { it.type == MarkdownTokenTypes.CODE_FENCE_END }
                        .filterNot { it.type == MarkdownTokenTypes.FENCE_LANG }
                        .dropWhile { it.type == MarkdownTokenTypes.EOL }
                        .dropLastWhile { it.type == MarkdownTokenTypes.EOL }
                        .mapNotNull { it.toContent(markdown) },
                ),
            )

        else -> error("Unsupported node type: ${this.type.name}")
    }

fun ASTNode.toContent(markdown: String): Content? =
    when (this.type) {
        MarkdownElementTypes.STRONG ->
            Content.Strong(
                contents = children
                    .drop(2) // drop ** or __
                    .dropLast(2) // drop ** or __
                    .mapNotNull { it.toContent(markdown) },
            )

        MarkdownElementTypes.CODE_SPAN ->
            Content.Code(
                contents = children
                    .drop(1) // drop `
                    .dropLast(1) // drop `
                    .mapNotNull { it.toContent(markdown) },
            )

        MarkdownTokenTypes.ATX_CONTENT ->
            Content.Text(text = getTextInNode(markdown).toString().trimStart())

        else ->
            Content.Text(text = getTextInNode(markdown).toString())
    }
```

</details>

```kotlin
sealed interface Block {
    sealed interface Text : Block { val contents: List<Content> }
    data class Paragraph(override val contents: List<Content>) : Text
    data class H1(override val contents: List<Content>) : Text
    data class Code(val contents: List<Content>) : Block
}

sealed interface Content {
    data class Text(val text: String) : Content
    data class Strong(val contents: List<Content>) : Content
    data class Code(val contents: List<Content>) : Content
}
```

## Composeによる描画

TODO: 説明

```kotlin
@Composable
fun Markdown(
    markdown: String,
    modifier: Modifier = Modifier,
) {
    Column(
        verticalArrangement = Arrangement.spacedBy(4.dp),
        modifier = modifier,
    ) {
        val markdownParser = remember { MarkdownParser(CommonMarkFlavourDescriptor()) }
        val blocks by produceState<List<Block>?>(
            initialValue = null,
            key1 = markdown,
            key2 = markdownParser
        ) {
            value = withContext(Dispatchers.IO) {
                markdownParser
                    .buildMarkdownTreeFromString(text = markdown)
                    .toBlocks(markdown = markdown)
            }
        }
        blocks?.forEach { block ->
            when (block) {
                is Block.Text -> TextBlock(block = block)
                is Block.Code -> CodeBlock(block = block)
            }
        }
    }
}

@Composable
private fun TextBlock(block: Block.Text) {
    Text(
        text = buildAnnotatedString {
            withStyle(
                style = when (block) {
                    is Block.Paragraph -> SpanStyle()
                    is Block.H1 -> SpanStyle(fontSize = 1.75.em)
                },
            ) {
                block.contents.forEach { content ->
                    append(content = content)
                }
            }
        },
    )
}

@Composable
private fun CodeBlock(block: Block.Code) {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .border(width = 1.dp, color = MaterialTheme.colorScheme.outlineVariant)
            .padding(4.dp),
    ) {
        Text(
            text = buildAnnotatedString {
                withStyle(style = SpanStyle(fontFamily = FontFamily.Monospace)) {
                    block.contents.forEach { content ->
                        append(content = content)
                    }
                }
            },
        )
    }
}
```

## このアプローチのメリット・デメリット

実装を見るとわかるように、`MarkdownParser` のパース以降の処理は完全に自らがハンドリングしています。そのため、以下のような恩恵を受けられます。

- 見出しのマージン、コードブロックのデザインなどを自由に決めることができる
- 特定のリンク（例：YouTubeのURL）は、単なるテキストリンクではなく、動画再生できるようにするなど、といった大胆な拡張も可能
- Markdownの機能を全て活かすには、画像の描画などを含めて多くの実装が必要

## 参考文献

- [JetBrains/compose-multiplatform | GitHub](https://github.com/JetBrains/compose-multiplatform)
- [JetBrains/markdown | GitHub](https://github.com/JetBrains/markdown)
