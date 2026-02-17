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

# SnapshotStateList の hashCode/equals に騙された

ここでは `SnapshotStateList` に注目して説明します。同様のことが `SnapshotStateSet` や `SnapshotStateMap` でも発生します。

Compose から参照する `List` を扱うとき `mutableStateListOf` によって生成される `SnapshotStateList` を使うことが多いと思います。この `SnapshotStateList` は `hashCode` と `equals` がリストの内容ではなく、インスタンスの参照を基にしているため、同じ内容のリストであっても異なるインスタンスであれば等しくないと判断されます。

## `List` とは

Kotlin の `List` は、要素の順序を保持するコレクションで `interface` として定義されています。そのため、`List` の実装クラスは複数存在します。

`List` の Doc を確認すると、

> - `List.toString` should return a string containing string representation of contained elements in exact same order these elements are stored within the list.
> - `List.equals` should consider two lists equal if and only if they contain the same number of elements and each element in one list is equal to an element in another list at the same index.
> - `List.hashCode` should be computed as a combination of elements' hash codes using the following algorithm:  
>   ```kotlin
>   var hashCode: Int = 1
>   for (element in this) hashCode = hashCode * 31 + element?.hashCode() ?: 0
>   ```

という記述があります。つまり、`List` の `equals` と `hashCode` はリストの内容に基づいています。

そのため `listOf` や `mutableListOf` で生成されたリストは、内容が等しければ `toString` も `equals` も `hashCode` も等しいです。

## `SnapshotStateList` とは

`SnapshotStateList` は監視とスナップショットを可能とした `MutableList` です。それを可能にするためか、`hashCode` と `equals` はリストの内容ではなく、インスタンスの参照を基にしています。

そのため、`mutableStateListOf` で生成されたリストは、内容が等しいとしても `toString` も `equals` も `hashCode` も等しくありません。

### `SnapshotStateList` のリストの内容を取得するために

`SnapshotStateList` は `toList()` を呼び出すことで、内容に基づいた不変（Immutable）な `List` を取得できます。

```kotlin
fun toList(): List<T>
```

## 動作確認

```kotlin
val list: List<String> = listOf("a", "b")
val snapshotStateList: SnapshotStateList<String> = mutableStateListOf("a", "b")

list.toString() // [a, b]
snapshotStateList.toString() // SnapshotStateList(value=[a, b])@88539910
snapshotStateList.toList() // [a, b]

list.hashCode() // 4066
snapshotStateList.hashCode() // 88539910
snapshotStateList.toList().hashCode() // 4066

list == snapshotStateList // true
snapshotStateList == list // false
snapshotStateList.toList() == list // true
```

## まとめ

- `List` の `toString` `equals` と `hashCode` は正しく実装するように指示している
- `SnapshotStateList` の `toString` `equals` と `hashCode` はリストの内容ではなく、インスタンスの参照を基にしている
- `SnapshotStateList` は `toList()` を呼び出すことで、内容に基づいた `List` を取得できる

## 参考

- [SnapshotStateList | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/runtime/snapshots/SnapshotStateList)
- [List | Core API – Kotlin Programming Language ](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.collections/-list/)
