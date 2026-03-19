---
title: Nav3は状態の保持と描画を分けて考えると設計しやすい
tags:
  - Android
  - Kotlin
  - Compose
  - JetpackCompose
  - Navigation3
private: false
updated_at: '2026-03-19T12:24:45+09:00'
id: c9cb9d8279c9bad687cc
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Jetpack Navigation 3（以下Nav3）は柔軟性が高い一方で、実装方針が1つに定まっているわけではありません。そのため、実運用ではアプリの要件に応じて設計する場面が多く、実装が複雑になりやすいです。

本記事では、Nav3において状態の保持と描画を分けて考えると、なぜ設計しやすくなるのかを説明します。

なお、本記事で紹介するアプローチは [`android/nav3-recipes/pull/242`](https://github.com/android/nav3-recipes/pull/242) にて、公式のレシピとしては採用されませんでした。これはアプローチ自体に問題があるのではなく、レシピとして適切ではなかったためだと考えています。しかし、Nav3の設計の自由度や仕組みを理解する上では有用な視点であるため、本記事で取り上げます。

## Nav3とは

Nav3はComposeのための新しい画面遷移の仕組みです。バックスタックの状態変化がUIに反映される宣言的な設計になっています。

Nav3を扱うときに利用することが多い `NavBackStack`、`rememberDecoratedNavEntries`、`NavDisplay` について簡単に説明します。

### `NavBackStack`

`NavBackStack` はバックスタックを管理するための型です。実体としては、`Serializable` な `SnapshotStateList<NavKey>` です。

つまり `NavBackStack` は、Nav3の遷移を扱うための高度な仕組みというより、Nav3の標準的なバックスタック表現の1つと考えるのがよいでしょう。

独自の画面遷移を設計する場合は、`NavBackStack` を使わずに独自実装しても構いません[^no-navkey-navbackstack]。具体例は後述の[遷移の設計は自由](#遷移の設計は自由)で説明します。

[^no-navkey-navbackstack]: `rememberDecoratedNavEntries` や `NavDisplay` は `NavKey` に依存しません。`NavKey` は `NavBackStack` のために用意されたインターフェースです。

```kotlin
@Serializable(with = NavBackStackSerializer::class)
public class NavBackStack<T : NavKey> : MutableList<T>, StateObject, RandomAccess
```

### `rememberDecoratedNavEntries`

`rememberDecoratedNavEntries` は `backStack: List<T>` をもとに `entryDecorators` と `entryProvider` を適用し、`List<NavEntry<T>>` を生成する関数です。

代表的な `entryDecorators` としては、`SaveableStateHolderNavEntryDecorator` や `ViewModelStoreNavEntryDecorator` があります。これらは内部で各画面に `SaveableStateHolder` や `ViewModelStore` を関連付けます。

ここで重要なのは、画面の状態（たとえば `ViewModel` や `SavedState`）の寿命は、この関数に渡されるリストに含まれている期間と一致するという点です。リストからキーが削除されると、対応する状態も破棄されます。

```kotlin
@Composable
public fun <T : Any> rememberDecoratedNavEntries(
    backStack: List<T>,
    entryDecorators: List<@JvmSuppressWildcards NavEntryDecorator<T>> = listOf(),
    entryProvider: (T) -> NavEntry<T>
): List<NavEntry<T>>
```

### `NavDisplay`

`NavDisplay` は、`entries: List<NavEntry<T>>` を受け取って描画するための関数です[^navdisplay-backstack]。`NavDisplay` は遷移状態を管理するのではなく、`entries` を描画する役割を担います[^on-back]。

[^navdisplay-backstack]: `NavDisplay` には `backStack: List<T>` を直接受け取り、内部で `rememberDecoratedNavEntries` を呼ぶAPIもあります。また `Scene` を明示的に扱うAPIもあります。本記事では「遷移状態をどう分離して設計するか」に焦点を当てるため、`entries` を直接扱う形で説明します。

[^on-back]: `NavDisplay` は描画だけでなく `onBack` にて戻る操作の消費も行います。

```kotlin
@Composable
public fun <T : Any> NavDisplay(
    entries: List<NavEntry<T>>,
    modifier: Modifier = Modifier,
    // ...
    onBack: () -> Unit,
)
```

## 保持と描画を分けて考える

実際のアプリ開発の場面では、1方向の単純な遷移だけでなく、タブやナビゲーションバーによる遷移も扱うことが多いです。このようなUIでは「画面としては表示されていないが、タブの状態は保持しておきたい」という要件が発生します。

例えば、以下のように遷移状態を `NavigationState` という独自のモデルで管理することを考えましょう。このように保持対象のバックスタックと描画対象のバックスタックを分けて扱うことで、非アクティブなタブの状態は保持しつつ、現在表示中のタブだけを描画できます。タブ切り替えやナビゲーションバーを伴う構成でも、この分離を意識すると設計しやすくなります。

このようにすると、`NavDisplay` に渡す `entries` は `activeBackStack` の内容のみになりますが、`rememberDecoratedNavEntries` には `inactiveBackStack` も渡されているため、裏にあるタブの状態（`ViewModel` など）は破棄されずに保持されます。

```kotlin
interface NavigationState {
    /** 非アクティブなバックスタック */
    val inactiveBackStack: NavBackStack<NavKey>

    /** アクティブなバックスタック */
    val activeBackStack: NavBackStack<NavKey>
}

@Composable
fun NavigationState.toDecoratedEntries(
    entryProvider: (NavKey) -> NavEntry<NavKey>,
): List<NavEntry<NavKey>> {
    // 全てのバックスタックを decorated した entries を生成する
    val allEntries = rememberDecoratedNavEntries(
        // 非アクティブ・アクティブに限らず全てのバックスタックを渡す
        backStack = inactiveBackStack + activeBackStack,
        entryProvider = entryProvider,
    )

    // アクティブなバックスタックに対応する entries のみを返す
    return allEntries.takeLast(activeBackStack.size)
}
```

## 遷移の設計は自由

前述の実装例から分かるとおり、遷移の設計には高い自由度があります。少なくとも以下の2点を満たせば、遷移状態の持ち方と描画の分け方をある程度自由に設計できます。

- 状態を保持したいバックスタックを `rememberDecoratedNavEntries` に渡す
- 画面を描画したい `NavEntry` 群だけを `NavDisplay` に渡す

たとえば、以下のような `NavNode` を定義し、木構造を探索して「保持対象の一覧」と「描画対象の `entries`」を導出できれば、より汎用的な遷移管理も可能になります。

```kotlin
sealed interface NavNode {
    class Leaf(val key: NavKey) : NavNode
    class Stack(val children: List<NavNode>) : NavNode
    class Select(var selected: NavNode, val children: Set<NavNode>) : NavNode
}
```

## まとめ

- Nav3では、状態の保持と描画を分けて考えると設計しやすい
- `rememberDecoratedNavEntries` には、状態保持する対象のバックスタックを全て渡す
- `NavDisplay` には、生成された `entries` の中から描画するものだけを渡す

## 参考文献

- [Navigation 3 | App architecture | Android Developers](https://developer.android.com/guide/navigation/navigation-3)
- [android/nav3-recipes: Implement common use cases with Jetpack Navigation 3](https://github.com/android/nav3-recipes)
