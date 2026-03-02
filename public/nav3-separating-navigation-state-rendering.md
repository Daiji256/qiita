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

Jetpack Navigation 3（以下 Nav3）は柔軟性が高いです。その一方で、1 つの実装方針が定まっているわけではなく、単一の正解がありません。そのため、実用段階で自分で実装することが多くなったり、複雑になりやすいです。

本記事では、Nav3 を扱うときための遷移・状態・描画を分離して管理する設計パターンを紹介します。

## Nav3 とは

Nav3 は Compose のために新しく実装された画面遷移のための仕組みで、宣言的画面遷移と呼べるような設計になっています。

Nav3 には主に `NavBackStack`、`rememberDecoratedNavEntries`、`NavDisplay` が登場します。

### `NavBackStack`

`NavBackStack` は back stack を管理するためのものです。`NavBackStack` は内部的には `Serializable` な `SnapshotStateList<NavKey>` にすぎません。

独自の遷移を設計する場合は、`NavBackStack` を使わずに独自に実装するのも良いでしょう[^no-navkey-navbackstack]。その例を[遷移は自由](#遷移は自由)にて紹介します。

[^no-navkey-navbackstack]: `rememberDecoratedNavEntries` や `NavDisplay` では `NavKey` を参照しません。つまり、`NavBackStack` を使わない場合は `NavKey` も不要になります。

```kotlin
@Serializable(with = NavBackStackSerializer::class)
public class NavBackStack<T : NavKey> : MutableList<T>, StateObject, RandomAccess
```

### `rememberDecoratedNavEntries`

`rememberDecoratedNavEntries` は `backStack: List<T>` を `entryDecorators` と `entryProvider` により `List<NavEntry<T>>` を生成するための関数です。

主要な `entryDecorators` として、`SaveableStateHolderNavEntryDecorator` や  `ViewModelStoreNavEntryDecorator` があります。これにより、画面ごとに `SaveableStateHolder` や `ViewModelStore` を紐付けることができます。

ここで注目すべきは、画面ごとの状態は `NavBackStack` でも `NavDisplay` でもなく、生成される `NavEntry` に属するという点です。これが、後述の設計を可能にします。

```kotlin
@Composable
public fun <T : Any> rememberDecoratedNavEntries(
    backStack: List<T>,
    entryDecorators: List<@JvmSuppressWildcards NavEntryDecorator<T>> = listOf(),
    entryProvider: (T) -> NavEntry<T>
): List<NavEntry<T>>
```

### `NavDisplay`

`NavDisplay` は `entries: List<NavEntry<T>>` を受け取って描画するための関数です[^navdisplay-backstack]。`NavEntry` をどのように扱うかは `Scene` の実装に依存しますが、基本的には最後尾の `NavEntry` を描画することが多いでしょう。

[^navdisplay-backstack]: `NavDisplay` には `backStack: List<T>` を受け取り内部で `rememberDecoratedNavEntries` を利用する簡易的な API もあります。また `Scene` を扱う API もあります。本記事では状態管理に注目するため、これらは紹介しません。

```kotlin
@Composable
public fun <T : Any> NavDisplay(
    entries: List<NavEntry<T>>,
    modifier: Modifier = Modifier,
    // ...
    onBack: () -> Unit,
)
```

## 遷移 / 状態 / 描画を分離する

本記事のアプローチは次の 3 つに分けて考えることです：

- 遷移は `NavigationState` として管理する
- 状態は `rememberDecoratedNavEntries` で保持する
- 描画は `NavDisplay` に任せる

TODO: 以下の実装例を利用して、遷移・状態・描画の分離の例を説明する。

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

前述の実装例を見て分かる通り、遷移の設計の自由度は非常に高いです。

`rememberDecoratedNavEntries` にて非アクティブを含む全ての back stack に対して状態保持する。そして、`NavDisplay` で扱うためのアクティブな entries をフィルタ・ソートして返す。これができれば、遷移の内部構造は自由であると言えます。

例えば、以下のような `NavNode` を定義してそこから back stack を得ることができれば、ナビゲーションバーやタブのような複雑な遷移構造にも対応した汎用的な画面遷移を設計できます：

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

- TODO: 箇条書きでまとめる

## 参考文献

- [Navigation 3 | App architecture | Android Developers](https://developer.android.com/guide/navigation/navigation-3)
- [android/nav3-recipes: Implement common use cases with Jetpack Navigation 3](https://github.com/android/nav3-recipes)
