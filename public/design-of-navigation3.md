---
title: Nav3による画面遷移の設計をどう考えるか
tags:
  - Android
  - Kotlin
  - Compose
  - JetpackCompose
  - Navigation3
private: false
updated_at: '2026-03-23T21:56:24+09:00'
id: abbbb034ff35d854e11a
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

Jetpack Navigation 3（以下Nav3）は、Composeによる状態管理を前提に構築された新しい画面遷移ライブラリです。高い柔軟性とカスタマイズ性を備えています。

この記事では、Nav3による画面遷移の設計について、個人的な結論[^current-conclusion]に至るまでの試行錯誤と、要件に応じた構成の選択基準を整理します。

[^current-conclusion]: 2026年3月23日時点での個人的な結論です。今後のアップデートや知見により変化する可能性があります。

## 要件に応じた設計の変化

Nav3はComposeの状態を前提とした疎結合なAPI群であり、[android/nav3-recipes](https://github.com/android/nav3-recipes)でも要件に応じた複数の実装アプローチが示されています。

特に、ナビゲーションバーなどによる画面切り替えの有無、ディープリンクの要件、`Scene` の扱いといった要素によって、適切な設計は大きく変わります。

## 試行錯誤

### 単純なバックスタック

`rememberNavBackStack` で生成したバックスタックをそのまま `NavDisplay` に渡す、最もシンプルな構成です。これは、[android/nav3-recipes - Basic DSL Recipe](https://github.com/android/nav3-recipes/tree/main/app/src/main/java/com/example/nav3recipes/basicdsl)で紹介されています。

- **向いている要件:** 直線的な画面遷移のみで構成されている
- **メリット:** 処理をNav3に一任できるため、実装が極めてシンプル
- **デメリット:** ナビゲーションバーなどの画面切り替えへの対応が困難[^scene-navigation-bar]

[^scene-navigation-bar]: `Scene` を活用し、スタックに含めたまま `onBack` を呼び出さないなどの工夫は可能です。

```kotlin
val backStack = rememberNavBackStack(ANavKey)
NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    entryProvider = entryProvider {
        entry<ANavKey> { /*...*/ }
        entry<BNavKey> { /*...*/ }
    },
)
```

### 複数バックスタック

ナビゲーションバーによる画面切り替えに対応する、複数のバックスタックを保持する構成です。トップレベルの選択状態に応じて `rememberDecoratedNavEntries` で生成したエントリーを切り替えます。これは、[android/nav3-recipes - Multiple back stacks recipe](https://github.com/android/nav3-recipes/tree/main/app/src/main/java/com/example/nav3recipes/multiplestacks)や[Migrate from Navigation 2 to Navigation 3](https://developer.android.com/guide/navigation/navigation-3/migration-guide)で紹介されています。

- **向いている要件:** トップレベルのバックスタック切り替えのみによる画面遷移で構成されている
- **メリット:** 状態保持を明示的に管理でき、タブ切り替えなどにも自然に対応可能
- **デメリット:** 「深い階層（利用規約など）からトップレベル（ホームなど）への遷移」などを実現するには独自の拡張が必要

```kotlin
val topLevel by remember { mutableStateOf(TopLevel1NavKey) }
val backStacks = setOf(TopLevel1NavKey, TopLevel2NavKey).associateWith { key ->
    rememberNavBackStack(key)
}

val entryProvider = entryProvider { /*...*/ }
val entries = backStacks.mapValues { (_, stack) ->
    rememberDecoratedNavEntries(
        backStack = stack,
        entryProvider = entryProvider,
    )
}

NavDisplay(
    entries = entries[topLevel] ?: emptyList(),
    onBack = { /*...*/ },
)
```

### 木構造による状態定義

より複雑な要件に応えるため、遷移状態を木構造（`NavNode`）で定義し、現在の活性ノードを保持するアプローチです[^navnode-trial-and-error]。詳細な実装例は[Daiji256/android-showcase - navigation](https://github.com/Daiji256/android-showcase/tree/a7086ca259cea695bf03435c92b63be20ea848c0/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/navigation)に記載しています。

[^navnode-trial-and-error]: `NavNode` の設計には多くの時間を費やしました。leaf / stack / selectといった役割の定義や、Navigate Upなど、機能の追加・削除を繰り返しました。

- **向いている要件:** ネストした遷移や、表示状態と遷移状態を分離して扱いたいケース
- **メリット:** 画面の表示・非表示に関わらず遷移状態を一元管理でき、複雑な制御に強い
- **デメリット:** 基盤の設計コストが高く、特異な要件に対して `NavNode` 自体の拡張を強いられる

```kotlin
class NavNode(val key: NavKey) {
    var currentChild: NavNode? by mutableStateOf<NavNode?>(null)
    val children: SnapshotStateSet<NavNode> = mutableStateSetOf()
}
```

```kotlin
val node: NavNode = // ...
val inactiveBackStack: List<NavKey> = // ...
val activeBackStack: List<NavKey> = // ...

val allEntries = rememberDecoratedNavEntries(
    backStack = inactiveBackStack + activeBackStack,
    entryProvider = entryProvider,
)

val entries = remember(allEntries, inactiveBackStack, activeBackStack) {
    allEntries.takeLast(activeBackStack.size)
}
```

### ドメイン特化の専用設計（現在の結論）

`NavNode` のような汎用化は自由度が高い半面、ディープリンクや複数画面の同時pop、未知の要件[^unexpected]に対しては状態管理が肥大化しがちです。これは実質的に「Nav3の上にNav2のような汎用APIを再構築している[^my-nav2]」と言えます。

[^unexpected]: 1画面内での `NavDisplay` の多重配置や、複雑な `Scene` 構成などが挙げられます。

[^my-nav2]: 実際には `NavNode` の設計はNav2とは異なります。ただ、「外部要件を幅広く吸収するための汎用的な画面遷移API」という点ではNav2と似た性質があります。

Nav3の真価は、巨大なAPIではなく「疎結合な最小限のAPI群」である点にあります。そのため、「汎用的な画面遷移APIを作るのではなく、アプリのドメイン（要件）に特化した専用の設計をその都度定義する」ことが、Nav3の最も素直な活用法であるという結論に至りました。

APIの使い分けや設計のポイントについては、[Nav3は状態の保持と描画を分けて考えると設計しやすい](https://qiita.com/Daiji256/items/c9cb9d8279c9bad687cc)で紹介しています。

- **向いている要件:** アプリ固有の遷移要件が強く、専用設計の方がドメインを素直に表現できるケース
- **メリット:** 余計な抽象化コストを抑え、複雑な固有要件にシンプルに対応できる
- **デメリット:** 汎用性がなく、遷移ロジックとドメインが密結合になる

## まとめ

Nav3による画面遷移は、実装者が設計を完全にコントロールできます。無理にNav2のような汎用APIを構築するのではなく、公式レシピが示すようにアプリの要件に合わせて状態設計を行うことで、Nav3の強みを最大限に活かすことができます。

## 参考文献

- [android/nav3-recipes | GitHub](https://github.com/android/nav3-recipes)
- [Android Developers Blog: Jetpack Navigation 3 is stable | Android Developers Blog](https://android-developers.googleblog.com/2025/11/jetpack-navigation-3-is-stable.html)
- [Migrate from Navigation 2 to Navigation 3 | App architecture | Android Developers](https://developer.android.com/guide/navigation/navigation-3/migration-guide)
- [Nav3は状態の保持と描画を分けて考えると設計しやすい | Qiita](https://qiita.com/Daiji256/items/c9cb9d8279c9bad687cc)
- [Daiji256/android-showcase - navigation | GitHub](https://github.com/Daiji256/android-showcase/tree/a7086ca259cea695bf03435c92b63be20ea848c0/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/navigation)
