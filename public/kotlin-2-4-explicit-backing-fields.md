---
title: newArticle001
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

# Explicit Backing Fieldsの仕組みと注意点（Kotlin 2.4.0）

Kotlin 2.4.0からexplicit backing fieldsがstableになりました[^no-args-ebf]。本記事では、これまでの「よくあるコード」がどう変わるのか、その仕組みと注意点について紹介します。

[^no-args-ebf]: コンパイラオプションの `-Xexplicit-backing-fields` が不要になりました。

## 今までの書き方とこれからの書き方

クラス内部ではmutable（可変）として扱うものの、外部に公開する際はimmutable（不変）として扱いたい場合、これまでは `_state` のようなプライベートプロパティを用意する実装パターンが一般的でした。

Explicit backing fieldsの登場により、これを `field` キーワードを用いて1つのプロパティで簡潔に表現できるようになります。

```kotlin
object Before {
    private val _state = MutableStateFlow("initial")
    val state: StateFlow<String> = _state
}

object After {
    val state: StateFlow<String>
        field = MutableStateFlow("initial")
}
```

## どのように便利なのか？

`After` のように書くことで、外部からは単なる読み取り専用の `StateFlow<String>` として見えますが、クラスの内部（宣言されたスコープ内）では、自動スマートキャスト（automatic smart cast）により、`MutableStateFlow` として扱うことができます。これにより、不要な `_state` プロパティの定義（ボイラープレート）を減らし、コードをすっきりと保つことができます。

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

このような異なるオブジェクトに変換したい場合は、`StateFlow` に限らずexplicit backing fieldsは利用できません。

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

TODO: 箇条書きで3つ程度

## 参考文献

- [Explicit backing fields](https://kotlinlang.org/docs/properties.html#explicit-backing-fields)
- [KEEP-0430-explicit-backing-fields.md](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0430-explicit-backing-fields.md)
- [What's new in Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html)
- [What's new in Kotlin 2.3.0](https://kotlinlang.org/docs/whatsnew23.html)
