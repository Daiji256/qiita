---
title: intellij-markdownとComposeでMarkdownを柔軟に描画する
tags:
  - Markdown
  - Kotlin
  - Compose
private: false
updated_at: '2026-07-04T09:22:47+09:00'
id: 1e75a22ee19fc5dc0c04
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

本記事では、ComposeでMarkdownを描画する方法について紹介します。既存のMarkdown描画ライブラリを使うのではなく、[intellij-markdown](https://github.com/JetBrains/markdown)によりMarkdownをパースし、Composeで描画するという手順を踏みます。

Markdownのパースをintellij-markdownに任せることで、多くの複雑な文字列処理から解放されます。一方で、パースして得られた構文木（AST）を「どのように描画するか」は自前で実装するため、アプリ独自のデザイン適用や拡張などの自由度を確保できるのがこのアプローチの強みです。

## Markdownをパース

まずは `MarkdownParser` によりMarkdown形式の生の `String` をパースし、`ASTNode` を得ます。

```kotlin
val markdown = "# Header\nParagraph."
val markdownParser = MarkdownParser(CommonMarkFlavourDescriptor())
val node: ASTNode = markdownParser.buildMarkdownTreeFromString(text = markdown)
```

## `ASTNode` を扱いやすい形に変換

`ASTNode` にはMarkdownの強調表示やコード表示などがネストされた状態で格納されていますが、描画時に不要になる記号（`#` など）のトークンも含まれており、扱いには注意が必要です。

そこで、`ASTNode` を描画で扱いやすい独自のモデル（`Block` と `Content`）に変換します。このモデルはComposeの `Composable` や `AnnotatedString` にマッピングしやすいように設計しています。

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

`toBlocks` で得られた `List<Block>` をComposeで描画します。`Block` をComposable関数で扱い、`Content` は `AnnotatedString` で扱います。

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
        ) {
            value = withContext(Dispatchers.Default) {
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

private fun AnnotatedString.Builder.append(content: Content) {
    when (content) {
        is Content.Text ->
            append(text = content.text)

        is Content.Strong ->
            withStyle(style = SpanStyle(fontWeight = FontWeight.Bold)) {
                content.contents.forEach {
                    append(content = it)
                }
            }

        is Content.Code ->
            withStyle(style = SpanStyle(fontFamily = FontFamily.Monospace)) {
                content.contents.forEach {
                    append(content = it)
                }
            }
    }
}
```

## 描画サンプル

実装した `Markdown` の描画を確認してみます。

```kotlin
Markdown(
    markdown = """
        # Header

        Inline `code1`.
        **Strong Inline `code2`.**

        ```
        fun function(arg: Int): Int {
            return arg * 2
        }
        ```
    """.trimIndent(),
)
```

![Markdownの描画サンプル](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/1d1977ea-0fc1-4bf3-9d7d-a6b4c9c1112c.png)

## このアプローチのメリット・デメリット

実装を見るとわかるように、`MarkdownParser` によるパース以降の処理は完全に自らがハンドリングしています。そのため、以下のようなメリット・デメリットがあります。

### メリット

見出しのマージン、コードブロックの装飾などを自由に実装できます。また、特定のリンク（YouTubeのURLなど）は単なるテキストリンクではなく、動画プレイヤーなどのリッチなUIコンポーネントに差し替えるといった実装も可能です。

### デメリット

パース自体はintellij-markdownが行ってくれますが、Markdownのすべての機能（テーブル、画像、ネストされたリスト、引用など）をサポートするには、構文木の扱いとComposeのUI実装を自前で追加していく必要があります。

## 参考文献

- [JetBrains/compose-multiplatform | GitHub](https://github.com/JetBrains/compose-multiplatform)
- [JetBrains/markdown | GitHub](https://github.com/JetBrains/markdown)
