---
title: onNewIntentでsetIntent(intent)を呼ばないとgetIntent()は更新されない
tags:
  - Android
  - Activity
  - Intent
  - onNewIntent
private: false
updated_at: '2026-07-13T20:42:35+09:00'
id: 588fb991c56b01b57f22
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

`launchMode` を `singleTask` や `singleTop` に設定し、アプリ起動中に通知やディープリンクから `Activity` を再利用する設計はよく使われます。このとき、新しく届いたパラメータは `onNewIntent(intent)` の引数から受け取ることができます。

基本的にはこれで問題ありません。しかし、`onResume` などで別途 `intent`（または `getIntent()`）を呼び出す場合、ディープリンクを受け取った後でもアプリ初回起動時の古い `Intent` のままになっています。

本記事では、この `Activity` が保持する `Intent` を更新する方法を紹介します。

## なぜ `intent` が更新されないのか

`Activity` を再利用する時の仕様上、新しい `Intent` が届いたときに `onNewIntent` が呼ばれるだけで、`Activity` 内部が保持している `intent` は自動で更新されません。

そのため、ディープリンク等からアプリを再利用した後に、`onResume` などで別途 `intent` を参照しても、最初にアプリを起動したときと同じ情報が返ってきます。

## `intent` を更新する方法

`intent` を更新する方法はシンプルで、`onNewIntent` をオーバーライドし `setIntent(intent)` を実行することです。これにより、`Activity` が保持する `intent` が上書きされるため、以降どのタイミングで参照しても更新後の値を参照できます。

なお、Kotlin では `this.intent = intent` と記述することで、`setIntent()` が呼ばれます。

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

必ずしもすべてのケースで `setIntent(intent)` による更新が推奨されるわけではありません。例えば、以下のようなケースでは更新しない方が適切な場合があります。

- 初回起動時の経路を保持し続けたい場合
- 新しい `intent` は一時的なアクションのトリガーにすぎない場合

このように、仕様・要件に合わせて、更新するべきかを正しく使い分ける必要があります。

## まとめ

- `Activity` が再利用された際、自動で `getIntent()` の中身は更新されない
- `onNewIntent` 内で `setIntent(intent)` を実行することで、保持する `Intent` を更新できる
- 用途によっては更新しない方が良いケースもあるため、要件に合わせて判断する

## 参考文献

- [Activity | API reference | Android Developers](https://developer.android.com/reference/android/app/Activity)
- [Intents and intent filters | App architecture | Android Developers](https://developer.android.com/guide/components/intents-filters)
