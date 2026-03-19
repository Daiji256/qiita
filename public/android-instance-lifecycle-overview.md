---
title: Androidアプリにおけるインスタンスの生存期間とスコープ
tags:
  - Android
  - lifecycle
  - Hilt
private: false
updated_at: '2025-03-05T16:37:45+09:00'
id: c39f565eade4563ffa43
organization_url_name: null
slide: false
ignorePublish: false
---

本記事では、Androidアプリ開発において登場する各種オブジェクト（`Application`、`Android`、`ViewModel` など）のインスタンス生存期間とスコープについて整理する。各オブジェクトがどのタイミング・スコープで生成され、破棄されるのかを理解することで、状態管理やリソース管理の最適化、メモリリークの防止などにつながる。

なお、記述内容は正確を期すよう努めているものの、誤りが含まれる可能性がある。もしそのような点にお気づきの場合は、指摘していただきたい。

## 前提

本記事では、`androidx.navigation` とDagger Hiltの利用を想定して説明を進める。なお、スコープやインスタンスの生存期間の説明上、Dagger Hiltの文脈が登場するが、Dagger Hiltの依存性注入の解説は行わない。

### 取り扱うこと

- 各オブジェクトの概要
- 各オブジェクトの生存期間とスコープの概要

### 取り扱わないこと

- 各オブジェクトの詳細なライフサイクルの内部処理
- 実装方法や設計方針
- 各オブジェクトの詳細な説明

## 各オブジェクトのスコープについて

本記事で主に扱うオブジェクトは以下の通りである。

- `Application`：  
  `android.app.Application` のこと。
- `ActivityRetained`（`@ActivityRetainedScoped`）：  
  構成変更（画面回転など）の後も維持されるスコープのこと。Dagger Hiltにおけるスコープの一種であり、実際のオブジェクトとして明示的に存在するのではなく、DIコンテナとして管理される。
- `Activity`：  
  `android.app.Activity` のこと。
- `Screen`：`Activity` 下の遷移の管理がされている画面のこと。`Fragment` やComposeの `NavHost` の `composable` に当てはめてほしい。
- `ViewModel`：  
  `androidx.lifecycle.ViewModel` のこと。
- `Service`：  
  `android.app.Service` のこと。`Activity`（UI）とは独立してバックグランド処理に専念する。
- Saved Stateの仕組み：  
  `onSaveInstanceState` や `SavedStateHandle` などにより、一時的に状態を保存・復元する仕組み。

### オブジェクト間の親子関係

各オブジェクトは、以下のような親子関係で整理できる。

![{"Application":{"ActivityRetained":{"Activity":{"Screen":{}},"ViewModel":{}},"Service":{}},"Saved State":{}}](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/d09ab661-1ba8-4bf7-af33-e58d8a827e04.jpeg)

## 各オブジェクトの詳細

### `Application`

`Application` はプロセス起動の起点となるオブジェクトである。`Application` の起動と破棄はOSが管理するため、通常はアプリ内から直接制御することない。多くの場合、`Activity` や `Service` の起動要求に伴って `Application` が起動される。そして、`Activity` と `Service` が存在しないときにOSのリソース管理の判断により `Application` が破棄される（プロセスが終了する）。

また、設定アプリなどから、通知や位置情報などのパーミッションをOFFに切り替えたときにも `Application` は破棄される。

`Activity` よりも生存期間が長いため、`Activity` の破棄・再生成後にも値を維持できる。

![Applicationが持つするobject1はActivityが再生成された場合にも維持される。Activityが持っていたobject2は失う。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/92ba01f9-338b-487a-9a3c-b3f710556962.jpeg)

### `ActivityRetained`（`@ActivityRetainedScoped`）

`ActivityRetained` は、画面回転などの構成変更が発生した際にActivityが再生成されてもそのまま保持されるスコープである。Dagger Hiltにおけるスコープの一種であり、実際のオブジェクトとして明示的に存在するのではなく、DIコンテナとして管理される。

ただし、明示的に `Activity` を終了したとき（例：`activity.finish()`）やOSのリソース管理によって `Activity` が破棄されるときには、`ActivityRetained` によるリソースも破棄される。

主に、`ViewModel` の管理や、構成変更時に状態を維持するために活用される。

![構成変更により、Activityは再生成されるが、ActivityRetainedとViewModelは維持される。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/e8c39fb1-3226-4b18-870b-fb6f70d3b1dd.jpeg)

![明示的な、Activityの破棄では、ActivityRetainedとViewModelも破棄される。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/f6182649-4da6-47f0-8d2f-961d8d9dadea.jpeg)

### `Activity`

`Activity` は画面を扱うオブジェクトである。画面回転などの構成変更や、バックグラウンド時のOSの判断などで破棄される。

`Activity` の破棄によりUIに関する状態は失われるため、状態の保持・復元のためには、`Application`、`ViewModel`、Saved Stateなどを活用する必要がある。

### `Screen`

`Screen` は `Activity` 下の遷移の管理がされている画面のこと。`Fragment` やComposeの `NavHost` の `composable` に当てはめてほしい。画面遷移に連動して `ViewModel` の生存期間が管理される。

### `ViewModel`

`ViewModel` は、`ActivityRetained` のスコープに配置されることで、構成変更による再生成があっても、状態を維持する。

`Activity` や画面遷移のライフサイクルに連動した生存期間となる。具体的には、`Activity` や `Screen` が完全に破棄された（画面遷移のバックスタックから無くなった）ときに、`ViewModel` も破棄される。

注意すべきは、`Screen` が画面遷移のバックスタックにあり、表に出ていない場合にも `ViewModel` はアクティブである。

![Screen1 → Screen2に遷移した状態では、Screen1は非アクティブであるが、ViewModel1とViewModel2がアクティブである。Screen2をpopすると、Screen1がアクティブになり、ViewModel2は破棄される。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/392b258f-d23c-4541-bfde-b9f809142bb0.jpeg)

### `Service`

`Service` はバックグランドでの処理に用いら、UIとは独立して動作する。そのため `Application` は参照できるが、`Activity` や `ViewModel` は参照できない。

### Saved Stateの仕組み[^system-server]

[^system-server]: Saved Stateの仕組みは、保存するための領域が `Application` の外側にあるというだけの認識で説明を進める。`system_server` や `Bundle` などの技術的な解決策や仕組みの説明は割愛する。

`onSaveInstanceState` や `SavedStateHandle` などにより、一時的に状態を保存・復元する仕組みのこと。保存先の領域は `Application` のスコープ外にある。

`Activity` や `ViewModel` の状態を保存・復元されるために利用される。また、画面遷移などの複数の `Screen` 等をまたいだあたいのやり取りにも利用される。

![Activityの再生成後にもSaved Stateは保持される。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/5fc8b617-c589-4a14-9998-305e2da48203.jpeg)

## 注意すべきこと

生存期間や役割の違いから、いくつか特に注意すべきポイントを以下に示す。

### `activity.finish()` では `Application` は破棄されない

例えば、アプリの退会時に状態を初期化し、アプリを終了したいとする。このとき `activity.finish()` が最も単純な方法に思えるであろう。しかし、これでは `Activity` が破棄されるだけで、`Application` は破棄されるとは限らない。`Application` のスコープ（Dagger Hiltでの `Singleton`）で保持している状態は継続して生存する。

`activity.finish()` では `Activity` を終了させることはできるが、`Application` を終了させることができないことに注意する必要がある。

![Activityの終了により、Activityが保持するobject2は破棄されるが、Applicationが保持するobject1 は破棄されない。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/9cd360a9-3428-4d20-bcf6-e819354d208b.jpeg)

### `ViewModel` と `Activity` の生存期間の違いによるメモリリーク

`Activity` は構成変更により再生成される一方、`ViewModel` はそのスコープにより、長い生存期間を持つ。もし `ViewModel` に `Activity` の `context` などを渡してしまうと、`ViewModel` が参照し続けて、メモリが解放されない（メモリリークする）。

`Activity` だけでなく、`Fragment` や `Composable` においても同様にメモリリークが発生する。そのため、`ViewModel` 内では `ApplicationContext` のみを参照するなどの対策が必要である。

![ViewModelがActivityからcontextを受け取る。Activityの再生後もcontextを保持し続けるため、メモリが解放されない。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/f6c18d66-feb9-4690-b230-5d941b8c7a41.jpeg)

### Saved Stateと `Application` の不整合

例えば、入力画面でユーザーが値を入力し、その値を `Application` の領域に保存した場合を考える。入力後に確認画面に遷移し、その画面で値を参照する。このとき設定アプリを開き何らかのパーミッションをOFFにすると、`Application` が破棄され、入力した値も破棄される。アプリを再度開くと、Saved Stateにより確認画面を開いた状態に復元されるが、`Application` の値は復元されない。そのため、確認画面にいるが、入力値が存在しないという状態が生まれてしまう。

`Application` よりSaved Stateの生存期間が長いことにより、アプリ全体の状態に不整合が起きる危険性がある。それぞれに保存する値に注意する必要がある。

![パーミッションをOFFにすることで、Applicationの値が破棄されるが、Saved Stateは破棄されない。両方が保持されていることを前提としている場合、不整合が生じる。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/b3310ec3-f9f9-4de8-8c57-1156c68d5383.jpeg)

## まとめ

本記事では、Androidアプリにおける各種オブジェクトの生存期間とスコープについて整理した。

各オブジェクトのスコープやライフサイクルの理解は、Androidアプリケーションの設計・実装において非常に重要である。リソースの無駄な消費や状態の不整合を回避するためには、スコープやライフサイクルを理解する必要がある。

## 参考

1. [The activity lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)
2. [Request runtime permissions](https://developer.android.com/training/permissions/requesting)
3. [Save UI states](https://developer.android.com/topic/libraries/architecture/saving-states)
4. [App startup time](https://developer.android.com/topic/performance/vitals/launch-time)
5. [Dependency injection with Hilt](https://developer.android.com/training/dependency-injection/hilt-android)
6. [Use Hilt with other Jetpack libraries](https://developer.android.com/training/dependency-injection/hilt-jetpack)
