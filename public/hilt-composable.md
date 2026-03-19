---
title: EntryPointを使ったComposable関数への依存性注入with Hilt & Compose
tags:
  - Android
  - Kotlin
  - DI
  - JetpackCompose
  - Hilt
private: false
updated_at: '2025-07-12T10:39:24+09:00'
id: 2a9b4690f209db1fe878
organization_url_name: null
slide: false
ignorePublish: false
---

Composable関数内で、Hiltによって `@Provides` や `@Binds` で提供されたオブジェクトを直接的に扱いたい場面があります。

通常、Hiltを用いた依存性注入はViewModelを経由して行います。たとえば `@HiltViewModel` に依存性注入し、ComposableからそのViewModelを利用することで間接的に依存オブジェクトへアクセスします。

しかし、以下のようなケースではViewModelを使わずに依存オブジェクトを直接取得したい場合があります：

- Composable内にViewModelの概念を持ち込みたくない
- トーストやAnalytics、ログ出力などの処理を、シンプルにその場で呼び出したい
- 単一のComposableに対して一時的に依存オブジェクトを使用したい

本記事では、Composable関数から直接Hiltの依存オブジェクトを取得する方法を紹介します。

## エントリポイントとは

HiltはComposableに対して直接的なDIを提供していません。そのため、Hilt管理下のオブジェクトを取得するには「エントリポイント（Entry Point）」を定義する必要があります。

エントリポイントとは、`@Inject` が使用できない場所（Composable関数や主要なAndroidのクラス以外）から、Hiltが管理する依存オブジェクトへアクセスするためのインターフェースです。

エントリポイントは次のように、`@EntryPoint` と `@InstallIn` を使って定義します：

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface FooEntryPoint {
    val foo: Foo
}
```

エントリポイントは、`EntryPointAccessors` を使って取得します。対象のコンポーネントがどこにインストールされているかに応じて、使用する `Context` が変わります。たとえば `SingletonComponent` にインストールされている場合、`ApplicationContext` を使用します。

つまり、コンポーネントに合った `Context` があれば、任意のタイミングでHiltの依存オブジェクトへアクセスできます：

```kotlin
val entryPoint = EntryPointAccessors.fromApplication<FooEntryPoint>(applicationContext)
val foo = entryPoint.foo
```

## Composable内でエントリポイントを取得する

Composable関数では `LocalContext.current` によって `Context` を取得できます。それにより、Composable関数内でエントリポイントを取得し、依存オブジェクトにアクセスできます：

```kotlin
@Composable
fun rememberFoo(): Foo {
    val context = LocalContext.current
    return remember {
        EntryPointAccessors.fromApplication<FooEntryPoint>(context.applicationContext).foo
    }
}
```

複数のエントリポイントを扱う場合は、以下のように汎用的な関数を定義しておくと便利です：

```kotlin
@Composable
inline fun <reified T : Any> rememberSingletonEntryPoint(): T {
    val context = LocalContext.current
    return remember {
        EntryPointAccessors.fromApplication<T>(context.applicationContext)
    }
}
```

## Composableにスコープを定義する

アプリ全体で共有する `SingletonComponent` ではなく、Composable専用のスコープで依存関係を管理したい場合もあります。

そのような場合は、独自のスコープとコンポーネントを `@DefineComponent` を使って定義します。以下は、`ActivityComponent` の子として `ComposableComponent` を定義する例です：

```kotlin
@Scope
annotation class ComposableScoped

@ComposableScoped
@DefineComponent(parent = ActivityComponent::class)
interface ComposableComponent

@DefineComponent.Builder
interface ComposableComponentBuilder {
    fun build(): ComposableComponent
}
```

この場合、エントリポイントの取得には `Activity` が必要になります。Compose環境では `LocalActivity.current` により取得できます：

```kotlin
@Composable
inline fun <reified T : Any> rememberComposableEntryPoint(): T {
    val activity = checkNotNull(LocalActivity.current)
    return remember {
        val builderEntryPoint =
            EntryPointAccessors.fromActivity<ComposableComponentBuilderEntryPoint>(activity)
        EntryPoints.get(builderEntryPoint.builder.build(), T::class.java)
    }
}

@EntryPoint
@InstallIn(ActivityComponent::class)
interface ComposableComponentBuilderEntryPoint {
    val builder: ComposableComponentBuilder
}
```

## まとめ

- Composable関数から直接Hiltの依存オブジェクトを扱うには、エントリポイントを定義します
- `EntryPointAccessors` を使い、スコープに応じた `Context` を用いることでエントリポイントを取得できます
- 必要に応じて独自スコープ（`ComposableScoped`）とコンポーネント（`ComposableComponent`）を定義することで、より細かい依存関係の制御が可能です

## 参考文献

https://dagger.dev/hilt/entry-points.html

https://dagger.dev/api/latest/dagger/hilt/DefineComponent.html

https://developer.android.com/training/dependency-injection/hilt-android

https://github.com/Daiji256/android-showcase
