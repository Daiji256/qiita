---
title: Nav3による画面遷移の設計をどう考えるか
tags:
  - Android
  - Kotlin
  - Compose
  - JetpackCompose
  - Navigation3
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Jetpack Navigation 3（以下Nav3）は、Jetpack Composeによる状態管理を前提にゼロから構築された、新しい画面遷移ライブラリです。高い柔軟性とカスタマイズ性を持つことが大きな特徴の一つです。

この記事では、Nav3による画面遷移の設計について、個人的な結論[^current-conclusion]に至るまでの試行錯誤を紹介します。あわせて、どのような要件でどの構成を選びやすいかも整理します。

[^current-conclusion]: あくまでも2026年3月23日時点での個人的な結論です。今後のアップデートや知見により、考えを改める可能性は十分にあります。

## 要件で設計が変わるか

Nav3はComposeの状態を前提にした疎結合なAPI群で構成されており、[android/nav3-recipes](https://github.com/android/nav3-recipes)では要件に応じた実装方法が示されています。

特に、「ナビゲーションバーなどの画面を切り替えの有無」「ディープリンクの有無」「`Scene` の扱い」によって扱いやすい設計は変わります。

## 試行錯誤

### 単純なバックスタック

最初に試したのは、`rememberNavBackStack` で生成したバックスタックをそのまま `NavDisplay` に渡す構成です。`entryProvider` には `entryProvider` DSLを利用しました。

この実装は、[android/nav3-recipes - Basic DSL Recipe](https://github.com/android/nav3-recipes/tree/main/app/src/main/java/com/example/nav3recipes/basicdsl)で紹介されているアプローチと同様です。

- **向いている要件:** 直線的な画面遷移で構成されている
- **メリット:** 処理のほぼすべてをNav3に一任できるため、非常にシンプルに実装できる
- **デメリット:** ナビゲーションバーによる画面切り替えなど、「バックスタックからは外れるが、状態は保持したい」というケースへの対応が難しい[^scene-navigation-bar]

[^scene-navigation-bar]: `Scene` を活用し、バックスタックに含まれたまま `onBack` の処理をスキップさせるなどの工夫は可能です。

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

次に、ナビゲーションバーによる切り替えに対応するため、複数のバックスタックを保持する構成を試しました。それぞれのバックスタックから `rememberDecoratedNavEntries` を用いてエントリーを生成し、トップレベルの選択状態に応じて表示を切り替える手法です。

この実装は、[android/nav3-recipes - Multiple back stacks recipe](https://github.com/android/nav3-recipes/tree/main/app/src/main/java/com/example/nav3recipes/multiplestacks)や[Migrate from Navigation 2 to Navigation 3](https://developer.android.com/guide/navigation/navigation-3/migration-guide)で紹介されているアプローチと同様です。

- **向いている要件:** トップレベルのバックスタックの切り替えだけによる画面遷移だけ構成されている
- **メリット:** 状態保持を明示的に管理できるため、ナビゲーションバーによる切り替えなどに対応できる
- **デメリット:** 「利用規約からホーム（トップレベル）への遷移」などを実現するには、さらに独自の拡張が必要になる

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

「利用規約からトップレベルへの遷移」や「特定条件でのスタック切り替え」など、より柔軟な状態保持のニーズに応えるため、遷移状態を木構造で定義するアプローチを検討しました。具体的には、ノード（`NavNode`）を定義し、現在の活性ノードを保持する形です。

[^navnode-trial-and-error]: `NavNode` の設計には多くの時間を費やしました。leaf / stack / selectといった役割の定義や、Navigate Upなど、機能の追加・削除を繰り返しました。

`NavNode` を実用レベルにするにはSaveableへの対応や、状態遷移を型安全に操作するための関数定義が必要です。詳細な実装例は[Daiji256/android-showcase - navigation](https://github.com/Daiji256/android-showcase/tree/a7086ca259cea695bf03435c92b63be20ea848c0/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/navigation)で公開しています。

- **向いている要件:** トップレベル切り替えだけでは表現しづらいネストした遷移や、表示状態と遷移状態を分けて扱いたいケース
- **メリット:** 画面の表示・非表示に関わらず遷移状態を一元管理できるため、複雑な遷移にも柔軟に対応できる
- **デメリット:** 基盤部分の設計コストが高く、アプリ固有の要件により `NavNode` 自体の拡張を強いられる可能性がある

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

### ドメインに特化した専用設計（現在の結論）

`NavNode` を用いると、ある程度の汎用性と自由度を確保できます。そのため、多くのアプリの要件を満たせると考えています。一方で、ディープリンクからの特定の階層への直接復帰や、複数画面の同時popなどを汎用的に処理しようとすると状態管理が複雑化し、さらなる拡張が必要になります。また、現時点で想定できていない要件[^unexpected]に対しては、それに合わせた変更が必要になります。

[^unexpected]: 例えば、1画面内での `NavDisplay` の多重配置や、複雑な `Scene` 構成などが挙げられます。

ある意味で `NavNode` は、自由度をあえて制限することで「Nav3の上にNav2のような汎用APIを再構築している」状態に近いと言えます[^my-nav2]。

[^my-nav2]: 実際には `NavNode` の設計はNav2とは異なります。ただ、「外部要件を幅広く吸収するための汎用的な画面遷移API」という点ではNav2と似た性質があります。

そこで辿り着いたのが**「汎用的な画面遷移を設計するのではなく、アプリのドメイン（要件）に特化した専用の設計をアプリごとに定義する」**という考え方です。

Nav3の真価は、画面遷移のための巨大なAPIではなく、疎結合な最小限のAPI群で構成されている点にあります。これらを組み合わせることで、複雑な機能も構築可能です。もちろん、チームやプロダクトをまたいで共通化したい場合には一定の抽象化が有効な場面もありますが、少なくともアプリ固有の要件が強い場合には、アプリごとのニーズに合わせて最適な組み合わせを自ら定義することが、Nav3を有効に活用する方法の一つだと解釈しました。

具体的なAPIの使い分けや設計のポイントについては、[Nav3は状態の保持と描画を分けて考えると設計しやすい](https://qiita.com/Daiji256/items/c9cb9d8279c9bad687cc)で紹介しています。

- **向いている要件:** アプリ固有の遷移要件が強く、専用設計の方が素直なケース
- **メリット:** アプリ固有の複雑な要件に対して、余計な抽象化コストを抑えた実装が可能になる
- **デメリット:** アプリごとに遷移ロジックを設計する必要があり、設計とドメインが密結合になる

## まとめ

Nav3による画面遷移は、実装者がその設計を自由にコントロールできます。汎用的な画面遷移APIを定義することも可能ですが、android/nav3-recipesや公式ドキュメントが示すように、Nav3は要件に合わせて設計を行うことで、より有効に活用できます。

## 参考文献

- [android/nav3-recipes | GitHub](https://github.com/android/nav3-recipes)
- [Android Developers Blog: Jetpack Navigation 3 is stable | Android Developers Blog](https://android-developers.googleblog.com/2025/11/jetpack-navigation-3-is-stable.html)
- [Migrate from Navigation 2 to Navigation 3 | App architecture | Android Developers](https://developer.android.com/guide/navigation/navigation-3/migration-guide)
- [Nav3は状態の保持と描画を分けて考えると設計しやすい | Qiita](https://qiita.com/Daiji256/items/c9cb9d8279c9bad687cc)
- [Daiji256/android-showcase - navigation | GitHub](https://github.com/Daiji256/android-showcase/tree/a7086ca259cea695bf03435c92b63be20ea848c0/core/ui/src/main/kotlin/io/github/daiji256/showcase/core/ui/navigation)
