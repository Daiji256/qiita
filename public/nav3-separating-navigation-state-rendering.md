---
title: Nav3 で遷移 / 状態 / 描画を分離する設計パターン
tags:
  - Android
  - Kotlin
  - JetpackCompose
  - Compose
  - Navigation3
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# Nav3 で遷移 / 状態 / 描画を分離する設計パターン

## はじめに

Jetpack Navigation 3（以下 Nav3）は柔軟性が高い一方で、実装方針が 1 つに定まっているわけではありません。そのため、実運用ではアプリの要件に応じて設計する場面が多く、実装が複雑になりやすいです。

本記事では、Nav3 を扱うときにうやむやになることとの多い、遷移と状態について紹介します。

## Nav3 とは

Nav3 は Compose のための新しい画面遷移の仕組みです。従来の Navigation Compose とは異なり、宣言的な画面遷移となっています。

この記事では、`NavBackStack`、`rememberDecoratedNavEntries`、`NavDisplay` を扱います。

### `NavBackStack`

`NavBackStack` は back stack を管理するための型です。内部的には、`Serializable` な `SnapshotStateList<NavKey>` です。

つまり `NavBackStack` は、Nav3 の遷移を扱うための高度な仕組みというより、Nav3 の標準的な back stack 表現の 1 つと考えるのが良いでしょう。

独自の画面遷移を設計する場合は、`NavBackStack` を使わずに独自実装しても構いません[^no-navkey-navbackstack]。その例は後述の[遷移は自由](#遷移は自由)で説明します。

[^no-navkey-navbackstack]: `rememberDecoratedNavEntries` や `NavDisplay` は `NavKey` に依存しません。`NavKey` は `NavBackStack` のために用意されたインターフェースです。

```kotlin
@Serializable(with = NavBackStackSerializer::class)
public class NavBackStack<T : NavKey> : MutableList<T>, StateObject, RandomAccess
```

### `rememberDecoratedNavEntries`

`rememberDecoratedNavEntries` は `backStack: List<T>` をもとに `entryDecorators` と `entryProvider` を適用し、`List<NavEntry<T>>` を生成する関数です。

代表的な `entryDecorators` としては、`SaveableStateHolderNavEntryDecorator` や `ViewModelStoreNavEntryDecorator` があります。これにより、各画面に `SaveableStateHolder` や `ViewModelStore` をひも付けられます。

ここで重要なのは、画面ごとの状態は `NavBackStack` や `NavDisplay` ではなく、生成された `NavEntry` にひも付くという点です。

```kotlin
@Composable
public fun <T : Any> rememberDecoratedNavEntries(
    backStack: List<T>,
    entryDecorators: List<@JvmSuppressWildcards NavEntryDecorator<T>> = listOf(),
    entryProvider: (T) -> NavEntry<T>
): List<NavEntry<T>>
```

### `NavDisplay`

`NavDisplay` は、`entries: List<NavEntry<T>>` を受け取って描画するための関数です[^navdisplay-backstack]。`NavDisplay` の責務としては、遷移状態を管理するではなく、`entries` をどう描画するかです。

[^navdisplay-backstack]: `NavDisplay` には `backStack: List<T>` を直接受け取り、内部で `rememberDecoratedNavEntries` を呼ぶ API もあります。また `Scene` を明示的に扱う API もあります。この記事では「遷移状態をどう分離して設計するか」に焦点を当てるため、`entries` を直接扱う形で説明します。

```kotlin
@Composable
public fun <T : Any> NavDisplay(
    entries: List<NavEntry<T>>,
    modifier: Modifier = Modifier,
    // ...
    onBack: () -> Unit,
)
```

## 遷移と状態を分離する

back stack を直接引数に取る `NavDisplay` もあるくらい、Nav3 では back stack と状態の管理を曖昧にすることができます。しかし、実際のアプリではナビゲーションバーなど、back stack には含めないが状態を保持したい場面が多くあります。

そのため、遷移と状態をうまく分けて考えると、設計がしやすくなります。

例えば、以下のように、遷移状態を `NavigationState` というアプリ独自のモデルで管理します。`NavigationState` は、遷移履歴に含まれない非アクティブな back stack と、遷移履歴に含まれるアクティブな back stack を持っています。

TODO: もうちょっと説明する

```kotlin
class NavigationState(
    /** 遷移履歴に含まれない非アクティブな back stack */
    val inactiveBackStack: NavBackStack<NavKey>,

    /** 遷移履歴に含まれるアクティブな back stack */
    val activeBackStack: NavBackStack<NavKey>,
)

@Composable
fun NavigationState.toDecoratedEntries(
    entryProvider: (NavKey) -> NavEntry<NavKey>,
): List<NavEntry<NavKey>> {
    // 全ての back stack を decorated した entries
    val allEntries = rememberDecoratedNavEntries(
        // 非アクティブ・アクティブに限らず全ての back stack を渡す
        backStack = inactiveBackStack + activeBackStack,
        entryProvider = entryProvider,
    )

    // アクティブな back stack に対応する entries を返す
    return allEntries.takeLast(activeBackStack.size)
}
```

## 遷移は自由

TODO: もうちょっと説明を減らす

前述の実装例から分かる通り、遷移の設計自由度はかなり高いです。

本質的には、

1. 状態を保持したい画面集合を `rememberDecoratedNavEntries` に渡す
2. 実際に表示したい `NavEntry` 群を選んで `NavDisplay` に渡す

この 2 段階ができればよいため、**遷移の内部構造は `NavBackStack` そのものに縛られません**。

たとえば、以下のような `NavNode` を定義し、そこから「状態保持対象の一覧」と「表示対象の一覧」を導出できれば、ナビゲーションバーやタブのような複雑な遷移構造にも対応できます。

```kotlin
sealed interface NavNode {
    class Leaf(
        val key: NavKey,
    ) : NavNode

    class Stack(
        val children: List<NavNode>,
    ) : NavNode

    class Select(
        var selected: NavNode,
        val children: Set<NavNode>,
    ) : NavNode
}
```

## まとめ

- Nav3 では、遷移・状態・描画の責務を分離して考えると設計しやすい
- 遷移の内部構造は `NavigationState` のようなアプリ独自モデルとして管理できる
- 画面ごとの状態は `rememberDecoratedNavEntries` によって `NavEntry` にひも付けられる
- `NavDisplay` は表示対象の `entries` を描画する責務に寄せると見通しがよい

## 参考文献

- [Navigation 3 | App architecture | Android Developers](https://developer.android.com/guide/navigation/navigation-3)
- [android/nav3-recipes: Implement common use cases with Jetpack Navigation 3](https://github.com/android/nav3-recipes)
