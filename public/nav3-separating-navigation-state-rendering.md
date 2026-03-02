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

Jetpack Navigation 3（以下 Nav3）は柔軟性が高い。一方で、実用段階では複数バックスタックやネスト構造、ナビゲーションバーとの組み合わせにより、状態管理が複雑になりやすい。

本記事では、**遷移・状態・描画を分離して管理する設計パターン**を紹介する。公式ドキュメントと Nav3 Recipes を踏まえつつ、多くのケースで基礎になりうる構成を整理する。

## Nav3 の前提

Nav3 の特徴は明確である。

* back stack を開発者が直接操作する
* `NavDisplay` は `entries` を受け取って描画する
* 画面遷移は宣言的に表現する

`NavDisplay` のシグネチャは次の通りである。

```kotlin
@Composable
public fun <T : Any> NavDisplay(
    entries: List<NavEntry<T>>,
    modifier: Modifier = Modifier,
    onBack: () -> Unit,
)
```

ここで重要なのは、**`NavDisplay` は状態を生成しない**という点である。描画はあくまで `NavEntry` に依存する。

## rememberDecoratedNavEntries とは何か

Nav3 では `NavEntryDecorator` を用いて `NavEntry` に状態を装飾できる。`rememberDecoratedNavEntries` はそれをまとめて扱うための API である。

たとえば次のようなデコレータを渡せる。

* `SaveableStateHolderNavEntryDecorator`
* `ViewModelStoreNavEntryDecorator`

これにより、

* 画面ごとの `rememberSaveable`
* 画面単位の `ViewModel`

を `NavEntry` に紐付けられる。

重要なのは発想である。

> 状態は NavDisplay に属するのではなく、Entry に属する。

この分離が後述の設計を可能にする。

## 遷移 / 状態 / 描画を分離する

本記事のアプローチは次の三層で構成する。

* 遷移は `List<NavKey>` として管理する
* 状態は `rememberDecoratedNavEntries` で保持する
* 描画は `NavDisplay` に任せる

構造は単純である。

```
All Keys（遷移）
    ↓
rememberDecoratedNavEntries（状態）
    ↓
Decorated Entries
    ↓ filter
Active Entries
    ↓
NavDisplay（描画）
```

`NavDisplay` に渡すのは、状態を保持した `entries` のうち **active な subset** のみである。

## 実装例

最小構成は次のようになる。

```kotlin
@Composable
fun NavHost(
    allBackStacks: List<List<NavKey>>,
    activeStackIndex: Int,
    onBack: () -> Unit,
) {
    val allKeys = remember(allBackStacks) {
        allBackStacks.flatten()
    }

    val decoratedEntries = rememberDecoratedNavEntries(
        keys = allKeys,
        entryDecorators = listOf(
            SaveableStateHolderNavEntryDecorator(),
            ViewModelStoreNavEntryDecorator()
        )
    )

    val activeKeys = allBackStacks[activeStackIndex]
    val activeEntries = decoratedEntries.filter { it.key in activeKeys }

    NavDisplay(
        entries = activeEntries,
        onBack = onBack,
    )
}
```

ここでは、

* 保存する back stack の一覧を持つ
* それらをまとめて状態化する
* 表示は active な stack のみ行う

という流れになる。

## 遷移履歴上には存在させないが状態を保持したい場合

ナビゲーションバーを例にする。

タブ A → タブ B → タブ A と切り替えたとき、

* A のスクロール位置
* A のフォーム入力状態

を保持したい。

Nav2 では `popUpTo + saveState` によってこれを実現していた。しかし本質は API ではなく、**「表示しないが状態は保持したい」**という要求である。

本記事の構成では、

* A のエントリは `All Keys` に残す
* ただし `NavDisplay` には渡さない

という形で自然に実現できる。

状態と描画を分離しているため、遷移履歴と表示履歴を一致させる必要はない。

## 遷移の仕組みは自由である

重要なのは次の二点である。

* 保存する back stack の一覧を定義できること
* そこから active な subset を取り出せること

これができれば、遷移の内部構造は自由である。

単純な `List<NavKey>` でもよいし、より構造化されたモデルでもよい。

たとえば次のようなノード構造も定義できる。

```kotlin
sealed interface NavNode<T : NavKey> {
    val key: T

    class Leaf<T : NavKey>(
        override val key: T,
    ) : NavNode<T>

    class Stack<T : NavKey>(
        override val key: T,
        children: List<NavNode<T>>,
    ) : NavNode<T>

    class Select<T : NavKey>(
        override val key: T,
        selected: T,
        val children: Set<NavNode<T>>,
    ) : NavNode<T>
}
```

最終的に `NavEntry` の集合へ flatten でき、active な subset を抽出できるなら、この設計は成立する。

deep link や navigateUp も、遷移モデル側で解決すればよい。状態管理と描画には影響しない。

## まとめ

Nav3 に正解はない。しかし、

* 遷移
* 状態
* 描画

を明確に分離することで、多くの複雑なケースに対応できる。

`rememberDecoratedNavEntries` を中心に据え、`NavDisplay` へは active な `entries` のみを渡す。この構成は実用的かつ拡張性が高い。

状態を Entry に属させる設計は、Nav3 を使いこなす上で重要な視点である。

## 参考文献

* Android Developers: Navigation 3
  [https://developer.android.com/guide/navigation/navigation-3](https://developer.android.com/guide/navigation/navigation-3)

* Android Developers: NavEntryDecorator
  [https://developer.android.com/guide/navigation/navigation-3/naventrydecorators](https://developer.android.com/guide/navigation/navigation-3/naventrydecorators)

* Android nav3-recipes
  [https://github.com/android/nav3-recipes](https://github.com/android/nav3-recipes)

## 脚注

[^1]: `NavDisplay` には `backStack: List<T>` を直接受け取る簡易的な API も存在する。また `Scene` を扱う拡張版もある。本記事では状態管理を自前で行うため、それらは扱わない。

[^2]: `NavKey` の identity 設計は重要である。同一キーをどの粒度で一意とみなすかにより、状態の再利用単位が決まる。

[^3]: `allBackStacks.flatten()` のような単純実装は説明のためのものであり、実際には差分更新やキー安定性を考慮する必要がある。

[^4]: 状態が不要になったキーは `All Keys` から除去すればよい。これにより対応する `NavEntry` の状態も破棄される。
