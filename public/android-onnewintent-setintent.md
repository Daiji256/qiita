---
title: newArticle001
tags:
  - Android
  - Intent
  - onNewIntent
  - Activity
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# onNewIntentでsetIntent(intent)を呼ばないとgetIntent()は更新されない

## はじめに

`launchMode` を `singleTask` などに設定することで、起動中にディープリンク等 `Activity` を再利用するアプリはよくあります。このとき、`onNewIntent` の `intent` からディープリンクで起動した時に受け取れるリンクなどを受け取ります。

基本的には問題はありませんが、`onResume` などで `intent`（`getIntent()`）を呼び出したときにディープリンクを受け取ったあとでもアプリ起動時のものであることに気がつきました。

この `getIntent()` で取得できる `Intent` を更新する方法を紹介します。

## なぜ必要なのか？

Androidの仕様上、再利用された `Activity` に新しい `Intent` が届いても、`onNewIntent` が呼ばれるだけで `Activity`内部で保持する `intent` は（`getIntent()`の参照先）は自動で上書きされません。

そのため、ディープリンクを開くなどして `Activity` を再利用した後でも、`onResume` などで `intent` を参照した時、ディープリンク起動時の `Intent` ではなく、起動時の `Intent` になります。

## `intent` を更新する方法

`onNewIntent` をオーバーライドし、`setIntent(intent)` を実行して、`Activity` が保持する `intent` を更新します。

```kotlin
class MainActivity : ComponentActivity() {
    // ...

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        this.intent = intent // setIntent(intent)
    }
}
```

## 注意

必ずしも `onNewIntent` で `setIntent(intent)` を呼び出すことを推奨しているわけではありません。`onNewIntent` が呼ばれる条件はアプリによって様々です。場合によっては一時的な処理や無視だけで十分で、`intent` の更新は不要な場合があります。

アプリの仕様・要件に合わせて、正しくハンドリングする必要があります。

## まとめ

- `Activity` が再利用された際、自動で `intent` は更新されない
- `onNewIntent` で `setIntent(intent)` を実行することで、`intent` を更新できる

## 参考文献

- [Activity | API reference | Android Developers](https://developer.android.com/reference/android/app/Activity)
- [Intents and intent filters | App architecture | Android Developers](https://developer.android.com/guide/components/intents-filters)
