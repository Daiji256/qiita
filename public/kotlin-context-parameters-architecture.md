---
title: newArticle001
tags:
  - Kotlin
  - 設計
  - ContextParameters
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# Kotlinのcontextによる再利用性制限を意識した設計

## はじめに

ソフトウェア開発において、関数やプロパティの再利用性は重要視されることがあります。しかし、過度に汎用性が高いコンポーネントは、想定外の文脈で再利用されてしまうという誤用のリスクを伴います。

本記事では、Kotlin 2.4.0で安定化された言語機能であるcontext parameters（コンテキストパラメータ）を活用し、あえて関数の再利用性を制限することによる設計上のメリットについて紹介します。

## Context parametersとは

Context parametersは、関数やプロパティが実行されるために周囲のスコープ（環境）に特定の型が存在することを静的に要求できる機能です。通常の引数は呼び出し元で明示的に値を渡す必要がありますが、`context` を使えばスコープ内に存在する依存関係を暗黙的に解決できます。

```kotlin
interface UserService {
    fun getUser(): User?
    fun setUser(user: User)
}

interface Logger {
    fun log(message: String)
}

// 実行にはUserServiceとLoggerが必要
context(userService: UserService, logger: Logger)
fun login(user: User) {
    if (userService.getUser() == null) {
        userService.setUser(user = user)
        logger.log(message = "$user logged in")
    }
}
```

## 再利用性を制限する

どこからでも呼び出せる汎用的な関数は便利ですが、すべての関数を汎用的にすべきではありません。用途や文脈を無視した秩序のない呼び出しは、意図しない利用が増える原因となり将来的な保守性の低下につながります。

`context` を用いると、特定の関数を呼び出すためのスコープを型として強制できます。たとえば、`UserService` が存在する文脈でのみ実行を許可したい関数には、コンテキストとして `UserService` を要求させます。これにより、その関数内で実際に `UserService` のメソッドを呼び出しているかどうかにかかわらず、`UserService` が提供されていないスコープからの不正な呼び出しをコンパイルエラーとして弾くことができます。

ファイルやモジュール分割などによる可視性制御と比べても、より柔軟かつ論理的に利用用途やスコープを宣言できるのが大きな強みです。

```kotlin
// 実行にはUserServiceが必要
context(_: UserService, logger: Logger)
fun outputTryLoginMessage(user: User) {
    logger.log(message = "try login $user")
}
```

## 引数の削減と関心事の分離

関数に必要なすべての値を通常の引数として渡すと、処理対象の値と実行環境という異なる関心事が混在してしまいます。これは役割を曖昧にします。また、めんどくさい引数のバケツリレーにつながります。

`context` を用いて環境の依存関係を分離・暗黙化させることで、中間の関数は自分が使わない引数を引き回す必要がなくなります。

```kotlin
context(userService: UserService, logger: Logger)
fun login(user: User) {
    // outputTryLoginMessageに必要なUserServiceとLoggerは暗黙的に伝播
    outputTryLoginMessage(user = user)

    if (userService.getUser() == null) {
        userService.setUser(user = user)
        logger.log(message = "$user logged in")
    }
}
```

## UIコンポーネントにおけるスコープ制限

この設計アプローチは、ComposeのUIコンポーネント開発でも役立ちます。

例えば、設定画面でのみ表示すべきボタンなどのコンポーネントがある場合、単なる `Boolean` などの汎用的な引数ではなく、設定画面全体の状態（`SettingsState`）を `context` で明示的に要求させます。コンポーネント内部で実際に利用する値がごく一部であっても、あえて要求する `context` の範囲を広げることで、他の画面からの呼び出しを抑制します。

```kotlin
// 設定画面からの利用を想定しているため、contextにSettingsStateを指定
context(settingsState: SettingsState)
@Composable  
fun UserCard(modifier: Modifier = Modifier) {
    // ...
}
```

## まとめ

- `context` により、関数やプロパティに対して引数とは異なるアプローチで環境や依存関係を渡すことができる
- 実行環境（`context`）とデータ（引数）を分離することで、コードの関心事を明確にできる
- コンポーネントに必要以上の `context` をあえて要求させることで、呼び出し可能な範囲を意図的に限定し、誤用を防ぐことができる

# 参考文献

- [Context parameters | Kotlin Documentation](https://kotlinlang.org/docs/context-parameters.html)
- [What's new in Kotlin 2.4.0 | Kotlin Documentation](https://kotlinlang.org/docs/whatsnew24.html)
