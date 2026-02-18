---
title: SnapshotStateList の onString / equals / hashCode の罠
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# SnapshotStateList の `onString` / `equals` / `hashCode` の罠

## はじめに

Compose で `List` を扱うとき、`mutableStateListOf` で生成する `SnapshotStateList` を使うことがよくあるでしょう。しかし `SnapshotStateList` の `onString` / `equals` / `hashCode` の挙動は通常の `List` と異なります。

本記事では、

- `List` の仕様
- `SnapshotStateList` の設計
- なぜ `onString` / `equals` / `hashCode` が異なるのか
- `SnapshotStateList` をどう扱えば良いか

を整理します。

※ 本記事では `SnapshotStateList` を例に扱いますが、同様の性質は `SnapshotStateSet` や `SnapshotStateMap` にもあります。

## `List` の仕様

`List` は要素の汎用的な順序付きコレクション（リスト）を扱うための型で、`interface` で定義されています。そのため、具体的な実装は複数存在し、自分でも実装できます。

リストとして扱うことができるよう、`List` のドキュメントには `onString` / `equals` / `hashCode` の仕様が明確に指示されています。

<details><summary>ドキュメントの原文</summary>

> - `List.toString` should consider two lists equal if and only if they contain the same number of elements and each element in one list is equal to an element in another list at the same index. Unlike some other equals implementations, List.equals should consider two lists equal even if they are instances of different classes; the only requirement here is that both lists have to implement List interface.
> - `List.equals` should consider two lists equal if and only if they contain the same number of elements and each element in one list is equal to an element in another list at the same index.
> - `List.hashCode` should be computed as a combination of elements' hash codes using the following algorithm:
>   ```kotlin
>   var hashCode: Int = 1
>   for (element in this) hashCode = hashCode * 31 + element.hashCode()
>   ```

</details>

- `toString` は、要素の格納と同じ順序で、各要素の文字列表現を含む文字列を返す
- `equals` は、2 つのリストの要素数・要素順・要素同士が等しい場合にのみ `true`、それ以外は `false` を返す
- `hashCode` は、各要素の `hashCode` の 31 倍加算結果を返す

つまり、`List` はインスタンスではなく要素が比較されます。

例えば、インスタンスやそのリストが構成されるまでの流れが異なる場合でも、要素として等しければ、`equals` は `true` を返します。

```kotlin
val list1 = mutableListOf("a", "b")
val list2 = mutableListOf("a").also { it.add("b") }

list1 == list2 // true
```

## `SnapshotStateList` とは何か

`SnapshotStateList` は `androidx.compose.runtime.snapshots` に含まれる Compose のための `MutableList` です。Composable 関数内での状態管理を意識して設計されており、状態の監視やスナップショットが可能です。

そのため `SnapshotStateList` は、いくつかの点において通常の `List` とは異なります。

### `toString` / `equals` / `hashCode` が要素ベースではない

`SnapshotStateList` は `equals` と `hashCode` をオーバーライドしていません。また、`toString` も独自の形式で出力されます。

`SnapshotStateList` は `equals` はインスタンスで比較される一方、通常の `List` は要素で比較されるため、両者を比較すると非対称な結果になることに注意が必要です。

以下のコードにて確認できます：

```kotlin
val list: List<String> = listOf("a", "b")
val snapshotStateList: SnapshotStateList<String> = mutableStateListOf("a", "b")

list.toString() // [a, b]
snapshotStateList.toString() // SnapshotStateList(value=[a, b])@88539910

list.hashCode() // 4066
snapshotStateList.hashCode() // 88539910

list == snapshotStateList // true
snapshotStateList == list // false
```

### なぜこのような設計なのか

`SnapshotStateList` は Composable で状態管理を行うためのコンテナであり、状態の変更を追跡することが主な目的です。つまり、`equals` の結果が変わってしまうと、状態の変更を正しく検知できなくなります。

そのため、例外的ではありますが、`SnapshotStateList` は要素の比較を行わない設計になっているではないかと推測されます。

## `SnapshotStateList` を `List` として `toString` / `equals` / `hashCode` を扱うには

`SnapshotStateList` は `toList()` を呼び出すことで、immutable な `List` を取得できます。この `List` は通常通りの `toString` / `equals` / `hashCode` の仕様を持ちます。

```kotlin
val snapshotStateList: SnapshotStateList<String> = mutableStateListOf("a", "b")

snapshotStateList.toList().toString() // [a, b]
snapshotStateList.toList().hashCode() // 4066
snapshotStateList.toList() == list // true
```

## まとめ

- `List` は、それが持つ要素により `toString` / `equals` / `hashCode` を定義している
- `SnapshotStateList` は、自身のインスタンスにより `toString` / `equals` / `hashCode` を定義している
- 要素としての比較が必要なら `toList()` を使う

## 参考

- [SnapshotStateList | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/runtime/snapshots/SnapshotStateList)
- [List | Core API – Kotlin Programming Language ](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.collections/-list/)
