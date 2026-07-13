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

`launchMode` を `singleTask` や `singleTop` に設定し、アプリ起動中に通知やディープリンクから `Activity` を再利用する設計はよく使われます。このとき、新しく届いたパラメータは `onNewIntent(intent)` の `intent` から受け取ることができます。

基本的にはこれで問題ないのです。しかし `onResume` などで `intent`（または `getIntent()`）を呼び出す場合、ディープリンクを受け取った後でもアプリ初回起動時の `intent` になります。

本記事では、この `getIntent()` で取得できる `Intent` を更新する方法を紹介します。

## なぜ `intent` が更新されないのか

Androidの仕様上、再利用された `Activity` に新しい `Intent` が届いても、`onNewIntent` が呼ばれるだけで、`Activity` 内部が保持している `intent` は自動で更新されません。

そのため、ディープリンク等からアプリを再利用した後に、`onResume` などで別途 `intent` を参照しても、最初にアプリを起動したときの古いものになります。

## `intent` を更新する方法

`intent` を更新する方法はシンプルで、`onNewIntent` をオーバーライドし `setIntent(intent)` を実行することです。これにより、`Activity` が保持する `intent` が更新されるため、以降どのタイミングで `intent` を参照しても呼び出しても、更新後の値を参照できます。

```kotlin
class MainActivity : ComponentActivity() {
    // ...

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        this.intent = intent // setIntent(intent) と同等
    }
}
```

## 無条件に更新すれば良いわけではない

必ずしもすべてのケースで `onNewIntent` での `setIntent(intent)` が推奨されるわけではありません。`onNewIntent` が呼ばれる条件はアプリによって様々です。場合によっては一時的な処理だけで十分で、`intent` の更新は不要な場合があります。

アプリの仕様・要件に合わせて、正しく使い分ける必要があります。

## まとめ

- `Activity` が再利用された際、自動で `getIntent()` の中身は更新されない
- `onNewIntent` で `setIntent(intent)` を実行することで、保持する `intent` を更新できる

## 参考文献

- [Activity | API reference | Android Developers](https://developer.android.com/reference/android/app/Activity)
- [Intents and intent filters | App architecture | Android Developers](https://developer.android.com/guide/components/intents-filters)
