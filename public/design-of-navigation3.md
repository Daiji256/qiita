---
title: newArticle001
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# Nav3による画面遷移の設計をどう考えるか

Jetpack Navigation 3（以下Nav3）はJetpack Composeによる状態管理を前提に、ゼロから構築された新しい画面遷移のためのライブラリです。高い柔軟性・カスタマイズ性を持つことが特徴の1つです。

この記事では、Nav3と向き合いいくつかの設計を考えた上で辿り着いた結論[^current-conclusion]までの「通ってきた道」を紹介します。

[^current-conclusion]: あくまでも、2026-03-23時点での結論です。今後、考えを改める可能性は十分にあります。

## 通ってきた道

### 単純なバックスタック

最初に試したのは、`rememberNavBackStack` で生成したバックスタックをそのまま `NavDisplay` に渡すという構成です。`entryProvider` は `entryProvider` DSLを利用しました。

この実装は[android/nav3-recipes - Basic DSL Recipe](https://github.com/android/nav3-recipes/tree/main/app/src/main/java/com/example/nav3recipes/basicdsl)で紹介されているアプローチと同等です。

- **メリット:** Nav3に処理のほぼ全てを任せることができるため、簡単に実装できる
- **デメリット:** ナビゲーションバーによる画面切り替えなどの「バックスタックからは消えるが状態は保持したいとき」の対応が難しい[^scene-navigation-bar]

[^scene-navigation-bar]: `Scene` をうまく使うことで、バックスタック上に含まれた上で `onBack` の処理が実行されないようにするなどの工夫は可能です。

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

ナビゲーションバーによる切り替えに対応するために、複数のバックスタックを用意する構成を試しました。それぞれのバックスタックから `rememberDecoratedNavEntries` を利用してエントリーを生成し、トップレベルの選択に応じて切り替える形です。

この実装は[android/nav3-recipes - Multiple back stacks recipe](https://github.com/android/nav3-recipes/tree/main/app/src/main/java/com/example/nav3recipes/multiplestacks)や[Migrate from Navigation 2 to Navigation 3](https://developer.android.com/guide/navigation/navigation-3/migration-guide)で紹介されているアプローチと同等です。

- **メリット:** 状態保持を自ら管理できるため、ナビゲーションバーによる切り替えなどに対応できる
- **デメリット:** 利用規約 → ホーム（トップレベル）などの遷移に対応するには、さらに拡張させる必要がある

```kotlin
val topLevel = remember { mutableStateOf(TopLevel1NavKey) }
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

### 木構造で状態を定義

利用規約からトップレベルへの遷移や、AとBの切り替えなど、柔軟なバックスタックの切り替えや状態保持が必要なケースは他にもあると考えました。そこで、遷移状態を木構造で定義しました。ここで説明するのは、その一例[^navnode-trial-and-error]でノード（`NavNode`）を定義し、現在のノードを保持する形です。

[^navnode-trial-and-error]: `NavNode` の試行錯誤には時間がかかりました。`NavNode` にleaf/stack/selectといった役割を持たせたり、navigate upのための機能を持たせたりなど、機能の追加・削除を繰り返しました。

`NavNode` による遷移を実用に耐えるものにするには、saveableにする対応や、うまく状態を書き換えるための関数の定義などが必要です。詳しい実装例は[Daiji256/android-showcase - navigation](https://github.com/Daiji256/android-showcase/tree/a7086ca259cea695bf03435c92b63be20ea848c0/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/navigation)にあります。

- **メリット:** 表示・非表示に関わらず、まとめて状態を管理できるため、ナビゲーションバーによる切り替えなどに対応できる
- **デメリット:** アプリの要件を満たすために、拡張を強いられる場合があり、うまく設計・実装する必要がある

```kotlin
class NavNode internal constructor(
    val key: NavKey,
) {
    var currentChild: NavNode? by mutableStateOf<NavNode?>(null)
    val children: SnapshotStateSet<NavNode> = mutableStateSetOf<NavNode>()
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

### ドメインに特化した専用の設計

`NavNode` による画面遷移はある程度の汎用性・自由度を確保します。そのため、多くのアプリの要件を満たすことができると考えています。一方で、ディープリンクやnavigate up（上へ移動）などの実装を考えると、さらなる拡張が必要になります。また、現時点で想定できていない要件[^unexpected]には、それに合わせた変更が必要です。

[^unexpected]: 例えば、`NavDisplay` を複数配置することや、複雑な `Scene` の構成などが考えられます。

言い換えると `NavNode` はある程度まで汎用性・自由度を制限することで、特定のドメインに縛られず、かつそこそこ安全に画面遷移を管理するためのものです。悪く言えば、Nav3を用いてNav2のような画面遷移APIを再構築しているような感覚があります[^my-nav2]。

[^my-nav2]: 実際には `NavNode` の設計はNav2とは異なります。ただ、「誰かが考えた汎用的な画面遷移API」という点ではNav2と同様です。

そこで、`NavNode` のように汎用的な画面遷移の設計を考えるのではなく、アプリのドメイン（要件）に特化した専用の設計をアプリごとに定義するという考え方に辿り着きました。

Nav3の大きな特徴は、画面遷移のための様々な機能を搭載した巨大なAPIではなく、いくつかの単純なAPIだけで構成された点です。これらを組み合わせることで複雑な機能も実装することができます。つまり、アプリごとのニーズに合わせて、うまく組み合わせることが、Nav3を最も有効に活用する方法です。

Nav3のそれぞれのAPIとそれに対して何を意識するかは、[Nav3は状態の保持と描画を分けて考えると設計しやすい](https://qiita.com/Daiji256/items/c9cb9d8279c9bad687cc)にて紹介しています。

- **メリット:** あらゆる要件に対応できる
- **デメリット:** アプリごとに画面遷移を設計する必要があり、遷移の設計とドメインが密結合になる

## まとめ

Nav3による画面遷移は、実装者が自由に設計できます。設計に汎用性を持たせることもできますが、Nav3を最大限に活用するにはアプリごとにそれぞれの設計を考えるとよいです。

## 参考文献

- [android/nav3-recipes | GitHub](https://github.com/android/nav3-recipes)
- [Android Developers Blog: Jetpack Navigation 3 is stable | Android Developers Blog](https://android-developers.googleblog.com/2025/11/jetpack-navigation-3-is-stable.html)
- [Migrate from Navigation 2 to Navigation 3 | App architecture | Android Developers](https://developer.android.com/guide/navigation/navigation-3/migration-guide)
- [Nav3は状態の保持と描画を分けて考えると設計しやすい | Qiita](https://qiita.com/Daiji256/items/c9cb9d8279c9bad687cc)
- [Daiji256/android-showcase - navigation | GitHub](https://github.com/Daiji256/android-showcase/tree/a7086ca259cea695bf03435c92b63be20ea848c0/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/navigation)
