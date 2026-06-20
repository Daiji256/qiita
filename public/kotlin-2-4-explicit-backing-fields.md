---
title: Explicit Backing Fieldsの仕組みと注意点（Kotlin 2.4.0）
tags:
  - Kotlin
  - ExplicitBackingFields
  - StateFlow
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Kotlin 2.4.0からexplicit backing fieldsがstableになりました[^no-args-ebf]。本記事では、これまでの「よくあるコード」がどう変わるのか、その仕組みと注意点について紹介します。

[^no-args-ebf]: `-Xexplicit-backing-fields` が不要になりました。

## 今までの書き方とこれからの書き方

`class` / `object` 内部ではmutable（可変）として扱うものの、外部に公開する際はimmutable（不変）として扱いたい場合、これまでは `_state` のようなprivateなpropertyを用意する実装パターンが一般的でした。

Explicit backing fieldsの登場により、これを `field` キーワードを用いて1つのプロパティで簡潔に表現できるようになります。

```kotlin
object Before {
    private val _state = MutableStateFlow("initial")
    val state: StateFlow<String> = _state
}
```

```kotlin
object After {
    val state: StateFlow<String>
        field = MutableStateFlow("initial")
}
```

## どのように便利なのか？

`After` のように書くことで、外部からは単なる読み取り専用の `StateFlow<String>` として見えますが、宣言されたスコープ内では、自動スマートキャスト（automatic smart cast）により、`MutableStateFlow` として扱うことができます。これにより、不要な `_state` の定義を減らし、コードをすっきりと保つことができます。

## どのような注意が必要か？

コードが簡潔になる一方で、引き続き注意する点もあります。それは、公開されている `state` が `StateFlow` のように見えているだけで、インスタンスとしては内部の `MutableStateFlow` と全く同じものであるという点です。

つまり、`After.state is MutableStateFlow<String>` は `true` になります。そのため、以下のように `login()` メソッドを介さずに、外部から無理やりキャストして値を書き換えることができます。

```kotlin
object UserStore {
    val user: StateFlow<String>
        field = MutableStateFlow("Guest")

    fun login() {
        user.value = "Alice"
    }
}

(UserStore.user as MutableStateFlow<String>).value = "Bob"
println("user: ${UserStore.user.value}") // user: Bob
```

外部から確実に操作できないようにするためには、単に「型として見えなくする」だけではなく、別のimmutable（不変）なオブジェクトに変換して公開するしかありません。

したがって、以下のように `asStateFlow()` を用いて別の読み取り専用オブジェクトを生成する用途では、依然として `_user` のような従来パターンが必要であり、explicit backing fieldsは利用できません。

この問題は `StateFlow` に限らず `List` などでも同様であり、異なるオブジェクトに変換して公開したい場合は、explicit backing fieldsは利用できません。

```kotlin
object UserStore {
    private val _user = MutableStateFlow("Guest")
    val user = _user.asStateFlow()

    fun login() {
        _user.value = "Alice"
    }
}
```

## まとめ

- `field` キーワードを使うことで、`_state` といったprivateなpropertyを定義する必要がなくなり、コードが簡潔になる
- スマートキャストにより実現されているため、スコープ内外で同一のオブジェクトが参照される
- 外部に公開する際に、厳密にimmutableにする必要がある場合は、`asStateFlow()` などを用いて従来通りに `_state` などが必要になる

## 参考文献

- [Explicit backing fields](https://kotlinlang.org/docs/properties.html#explicit-backing-fields)
- [KEEP-0430-explicit-backing-fields.md](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0430-explicit-backing-fields.md)
- [What's new in Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html)
- [What's new in Kotlin 2.3.0](https://kotlinlang.org/docs/whatsnew23.html)
