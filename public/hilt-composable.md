---
title: TODO
tags:
  - Android
  - Kotlin
  - DI
  - JetpackCompose
  - Hilt
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# Hilt で Composable に DI する

Composable 関数内で、Hilt によって `@Provides` や `@Binds` により提供されたオブジェクトを扱いたい場面があります。通常、Composable と Hilt を併用する際は ViewModel を通じて依存オブジェクトを取得します。たとえば `@HiltViewModel` によって依存関係を注入し、それを Composable から利用するといった形です。

たとえば、ロガーや Analytics 機能など、Composable 内外から利用する共通実装であるとします。この場合、次のような理由から ViewModel を経由したくない場合もあります：

- Composable 内に ViewModel の概念を持ち込みたくない
- 特定機能に対して、より直接的に依存させたい

本記事では、Composable 関数内で直接 Hilt による依存オブジェクトを扱う方法を紹介します。

## エントリポイントとは

Hilt は Composable に直接対応していないため、依存オブジェクトを取得するには「エントリポイント」を定義する必要があります。

エントリポイント（Entry Point）とは、Hilt の管理外にあるコードから、Hilt 管理下のオブジェクトにアクセスするためのインターフェースです。`@Inject` が使えない場所でも、エントリポイントを通じて依存関係を取得できます。

エントリポイントは `@EntryPoint` アノテーションを付与して定義します。これにより Hilt のオブジェクトグラフに登録され、管理外の領域から依存を取得できるようになります。

たとえば、`SingletonComponent` にインストールされたエントリポイントは次のように定義します：

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface FooEntryPoint {
    val foo: Foo
}
```

## Composable 内でエントリポイントを取得する

エントリポイントへのアクセスには `EntryPointAccessors` を使います。どのコンポーネントにインストールされているかによって、必要な `Context` が異なります。`SingletonComponent` の場合は `ApplicationContext` を用います。

Composable 内では、`LocalContext` を用いて取得した `ApplicationContext` によりエントリポイントへアクセスします[^remember]：

[^remember]: `remember` を使うことで、リコンポジション時にオブジェクトが再生成されるのを防ぎます。

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

## Composable にスコープを定義する

前節では `SingletonComponent` を使いましたが、Composable により近いスコープで依存管理を行いたい場合もあります。その場合は、独自のスコープを `DefineComponent` を用いて実装します。

`ActivityComponent` の子コンポーネントとして `ComposableComponent` を定義します：

```kotlin
@Scope
annotation class ComposableScoped

@ComposableScoped
@DefineComponent(parent = ActivityComponent::class)
interface ComposableComponent
```

この `ComposableComponent` の場合は エントリポイントの取得には `Activity` が必要です。

Composable 内から `ComposableComponent` にアクセスするには、以下のようにします：

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

@DefineComponent.Builder
interface ComposableComponentBuilder {
    fun build(): ComposableComponent
}

@EntryPoint
@InstallIn(ActivityComponent::class)
interface ComposableComponentBuilderEntryPoint {
    val builder: ComposableComponentBuilder
}
```

## まとめ

- Hilt のエントリポイントを用いれば、Composable 関数内で依存オブジェクトを直接取得可能
- `EntryPointAccessors` を使い、スコープに応じた `Context` を与えることでエントリポイントにアクセス可能
- 独自の Composable スコープを定義することで、より柔軟な依存管理が可能

## 参考文献

https://dagger.dev/hilt/entry-points.html

https://dagger.dev/api/latest/dagger/hilt/DefineComponent.html

https://developer.android.com/training/dependency-injection/hilt-android

https://github.com/Daiji256/android-showcase
