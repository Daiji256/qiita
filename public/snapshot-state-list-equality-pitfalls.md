---
title: SnapshotStateList の toString / equals / hashCode の罠
tags:
  - Android
  - Kotlin
  - JetpackCompose
  - List
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# `SnapshotStateList` の `toString` / `equals` / `hashCode` の罠

tags: Android, Kotlin, Jetpack Compose, List

## はじめに

Compose で `List` を扱うとき、`mutableStateListOf` で生成される `SnapshotStateList` を利用することは多いでしょう。

しかし `SnapshotStateList` の `toString` / `equals` / `hashCode` の挙動は、通常の `List` と同じ前提では扱えません。この違いを理解していないと、比較処理などで意図しない挙動に遭遇します。

本記事では、次の観点について解説します：

- Kotlin の `List` について
- `SnapshotStateList` の性質
- なぜ `toString` / `equals` / `hashCode` で差が出るのか
- `SnapshotStateList` の扱い方

※ 同様の注意点は `SnapshotStateSet` や `SnapshotStateMap` にもあります。

## `List` の仕様

`List` は要素の順序を保持するコレクション型であり、`interface` として定義されています。そのため、具体的な実装は複数存在し、独自実装も可能です。

Kotlin のドキュメントでは、`List` の `toString` / `equals` / `hashCode` について明確な記述があります。

要点は次の通りです：

- `toString` は要素を順序通りに並べた文字列表現を返す
- `equals` は要素数・順序・各要素の `equals` に基づいて比較する
- `hashCode` は各要素の `hashCode` を組み合わせて算出する

<details><summary>ドキュメントの原文</summary>

> - `List.toString` should return a string containing string representation of contained elements in exact same order these elements are stored within the list.
> - `List.equals` should consider two lists equal if and only if they contain the same number of elements and each element in one list is equal to an element in another list at the same index. Unlike some other `equals` implementations, `List.equals` should consider two lists equal even if they are instances of different classes; the only requirement here is that both lists have to implement `List` interface.
> - `List.hashCode` should be computed as a combination of elements' hash codes using the following algorithm:
>   ```kotlin
>   var hashCode: Int = 1
>   for (element in this) hashCode = hashCode * 31 + element.hashCode()
>   ```

</details>

つまり、`List` は構造的等価（structural equality）で比較されます。

生成過程が異なっていても、要素が同じであれば等しいと判定されます：

```kotlin
val list1 = mutableListOf("a", "b")
val list2 = mutableListOf("a").also { it.add("b") }

list1 == list2 // true
```

## `SnapshotStateList` とは何か

`SnapshotStateList` は `androidx.compose.runtime.snapshots` に含まれる Compose の状態管理向け `MutableList` 実装です。

この型は Snapshot システムと連携するため、通常の `List` 実装と違う挙動を持ちます。

### `toString` / `equals` / `hashCode` の違い

`SnapshotStateList` では、通常の `List` のような要素ベースの `toString` / `equals` / `hashCode`を持ちません。代わりに、インスタンスの参照をベースにした実装になっています。

また、Kotlin の `==` 演算子は、左辺の `equals` メソッドを呼び出すため、`SnapshotStateList` と通常の `List` を比較すると、一方は要素ベースで、もう一方は参照ベースで比較されることになります。

```kotlin
val list = listOf("a", "b")
val snapshotStateList = mutableStateListOf("a", "b")

list.toString()                 // [a, b]
snapshotStateList.toString()    // SnapshotStateList(value=[a, b])@88539910

list.hashCode()                 // 4066
snapshotStateList.hashCode()    // 88539910

list == snapshotStateList       // true
snapshotStateList == list       // false
```

### 通常の `List` のように扱うには

`SnapshotStateList` を要素ベースで扱いたい場合は、`toList()` を使って比較対象を通常の `List` にそろえるのが安全です。

`toList()` で取得した `List` は読み取り専用として扱え、`toString` / `equals` / `hashCode` も通常の `List` と同様に利用できます。また、取得後に `SnapshotStateList` 側を変更しても、すでに取得した `List` の内容は変わりません。

```kotlin
val snapshotStateList = mutableStateListOf("a", "b")
val immutableList = snapshotStateList.toList()

immutableList.toString()             // [a, b]
immutableList.hashCode()             // 4066
immutableList == listOf("a", "b")    // true
```

## まとめ

- 通常の `List` の `toString` / `equals` / `hashCode` は要素ベースで扱われる
- `SnapshotStateList` の `toString` / `equals` / `hashCode` は参照ベースで扱われる
- 要素比較が必要なら `toList()` により、通常の `List` と同様に扱う

## 参考

- [SnapshotStateList | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/runtime/snapshots/SnapshotStateList)
- [List | Core API – Kotlin Programming Language ](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.collections/-list/)
