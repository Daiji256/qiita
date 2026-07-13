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
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# Kotlinのcontextによる再利用性制限を意識した設計

## はじめに

関数やプロパティの再利用性は長らく重要視されることが多いです。しかし、汎用性が高いために、想定外の文脈での再利用されるという誤用のリスクを伴います。

本記事では、Kotlin 2.4.0で安定化したcontext parameters（コンテキストパラメータ）を活用し、あえて再利用性を制限することによる設計上のメリットについて紹介します。

## Context parametersとは

Context parametersはKotlin 2.4.0で安定化した言語機能です。これを使用することで、関数やプロパティは周囲のコンテキストで暗黙的に利用可能な依存関係を宣言できます。引数の場合は呼び出し元でも明示的に引数を渡す必要がありますが、`context` の場合は暗黙的にそれを解決できます。

```kotlin
interface UserService {
    fun getUser(): User?
    fun setUser(user: User)
}

interface Logger {
    fun log(message: String)
}

context(userService: UserService, logger: Logger)
fun login(user: User) {
    if (userService.getUser() == null) {
        userService.setUser(user = user)
        logger.log(message = "$user logged in")
    }
}
```

## 再利用性を制限する

どこからでも呼び出せる汎用的な関数は便利ですが、すべての関数を汎用的にするわけにはいきません。用途や文脈を無視した呼び出しが可能だと、意図しない利用により、変更が難しくなります。

`context` を用いると、特定の関数を呼び出すための前提条件を型として強制できます。たとえば、`UserService` をコンテキストとして持つ時のみに実行することを想定した関数に `UserService` を要求させます。これにより、`UserService` を利用するか・否かに限らず、`UserService` をコンテキストに持つスコープからしか呼び出せません。

これにより、ファイルやモジュールの分割に比べて柔軟に利用用途やスコープを宣言できます。

```kotlin
context(_: UserService, logger: Logger)
fun outputTryLoginMessage(user: User) {
    logger.log(message = "try login $user")
}
```

## 引数の削減と関心事の分離

関数に必要な値をすべて引数で渡すと、入力値と環境等という関心ごとが混在し、役割が曖昧になることやバケツリレーの発生などに繋がります。`context` に環境等を暗黙的に伝播させることで、中間の関数に不要な引数が不要になります。

```kotlin
context(userService: UserService, logger: Logger)
fun login(user: User) {
    outputTryLoginMessage(user = user)
    if (userService.getUser() == null) {
        userService.setUser(user = user)
        logger.log(message = "$user logged in")
    }
}
```

## UIコンポーネントにおけるスコープ制限

この設計アプローチは、Composeのコンポーネントでも役立ちます。例えば、設定画面でのみ表示するコンポーネントの場合、設定画面の状態（`SettingsState`）を `context` で明示します。実際に利用する値が一部でも、`context` を広くを持つことで、他画面からの再利用を拒否できます。

```kotlin
context(settingsState: SettingsState)
@Composable  
fun UserCard(modifier: Modifier = Modifier) {
    // ...
}
```

## まとめ

- `context` により、関数やプロパティに引数以外の方法で値を渡すことができる
- 必要以上に `context` を指定することで、関数を呼び出せる範囲を限定する

# 参考文献

- [Context parameters | Kotlin Documentation](https://kotlinlang.org/docs/context-parameters.html)
- [What's new in Kotlin 2.4.0 | Kotlin Documentation](https://kotlinlang.org/docs/whatsnew24.html)
