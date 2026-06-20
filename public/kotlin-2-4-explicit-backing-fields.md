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

Kotlin 2.4.0でstableになった（compiler argsに何もつかしなくて良くなった）Explicit Backing Fieldsについて紹介する

class内部でmutableとして扱うが、公開時はmutableとして扱いたくない時などは、`_state` のような実装パターンをしていたが、Explicit Backing Fieldsにより `field` で実現できるようになる：

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

`field` を使うことで内部ではsmart castにより、`MutableStateFlow` として扱える（smart castと呼んでいいのか自信がない、要調査）

公開されている `state` が `StateFlow` のように見えるだけで、インスタンスとしては同じなので、`After.state is MutableStateFlow<String>` は `true` になることに注意

つまり、以下の実装のように `login` を呼び出さずに、無理やり `value` を書き換えることができる：

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

外部から確実に操作できないようにするためには、別のimmutableなインスタンスに変換して公開するしかないため、依然として `_user` のようなパターンが必要で、Explicit Backing Fieldsは使えない：

```kotlin
object UserStore {
    private val _user = MutableStateFlow("Guest")
    val user = _user.asStateFlow()

    fun login() {
        _user.value = "Alice"
    }
}
```

実装例の `StateFlow` / `MutableStateFlow` に限らず、`List` などにおいても同様の問題がある

チーム内で扱うコードなどでは、そこまで気にする必要はないためExplicit Backing Fieldsは役立つだろうが、ライブラリ等で第三者に公開する場合などでは、immutableなものに変換する方が安全なのは変わらない
