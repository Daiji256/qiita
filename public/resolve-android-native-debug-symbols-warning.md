---
title: Google Play Consoleの「デバッグシンボルがありません」を解決する
tags:
  - Android
  - NDK
  - gradle
  - GooglePlayConsole
  - AndroidAppBundle
private: false
updated_at: '2026-06-14T02:12:36+09:00'
id: 1ba7d94767bedf4f254f
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Androidアプリをリリース・アップデートしようとした際、Google Play Console上で以下のワーニングが表示されたことはないでしょうか？

本記事では、Android App Bundle（`.aab` ファイル）にデバッグシンボルを含めるように設定し、この警告を解決する方法を紹介します。

> This App Bundle contains native code, and you've not uploaded debug symbols. We recommend you upload a symbol file to make your crashes and ANRs easier to analyze and debug.

> この App Bundle にはネイティブ コードが含まれ、デバッグ シンボルがアップロードされていません。クラッシュや ANR を簡単に分析、デバッグできるよう、シンボル ファイルをアップロードすることをおすすめします。

![Google Play Consoleに表示されるデバッグシンボル欠如の警告](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/7247fe64-8ede-45c0-abcb-e0403dd466c9.png)

## デバッグシンボルとは何か？

ソースコードをコンパイルしてバイナリ（マシン語）を作成する過程で、ファイルサイズの削減や最適化等により、変数名や関数名のようなソースコードの情報（デバッグ情報）が削除されます。

デバッグシンボルとは、このバイナリとソースコードを紐づけるためのマッピングデータです。

これがないと、そのバイナリ内での処理中にクラッシュが発生した際に、Play Console上からはどのようなエラーが起きたかの特定が困難になります。

## NDKをインストールする方法

まずは事前準備として、以下の手順によりAndroid StudioでNDKをインストールしておきます。

1. メニューからTools > SDK Managerを選択する
2. SDK Toolsタブを開き、Show Package Detailsにチェックを入れる
3. インストールしたいNDKのバージョンにチェックを入れる
4. Applyする

![Android StudioのSDK ManagerでのNDKインストール手順](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/7536cecc-8982-437d-a0fd-a32ed5716a01.png)

## 必要な設定

NDKの用意ができたら、以下のように `app/build.gradle.kts` に `ndkVersion` と `debugSymbolLevel`[^debug-symbol-level-full-symbol_table] を設定することで、ビルドされる `.aab` ファイル内にデバッグシンボルが同梱されるようになります。

[^debug-symbol-level-full-symbol_table]: `debugSymbolLevel` には `"SYMBOL_TABLE"` も指定可能ですが、Play Consoleでの詳細な解析には、関数名だけでなく詳細な行番号情報なども保持する `"FULL"` の指定が推奨されています。ただし、`"FULL"` を指定すると `.aab` のファイルサイズが `"SYMBOL_TABLE"` よりも大きくなります。

注意するべき点として、`ndkVersion` の明示的な指定が必要です。`debugSymbolLevel` だけを設定した場合、ビルド時にデバッグシンボル自体は生成されるものの、`.aab` には含まれませんでした。

```kotlin
android {
    ndkVersion = "27.3.13750724" // 執筆時点での最新LTSバージョン

    buildTypes {
        release {
            ndk.debugSymbolLevel = "FULL" // もしくは "SYMBOL_TABLE"
        }
    }
}
```

## 成功の確認方法

リリースビルドにて生成された `.aab` ファイルの拡張子を `.zip` に変更し解凍すると、 `BUNDLE-METADATA/com.android.tools.build.debugsymbols` が含まれているはずです。

実際にこの `.aab` ファイルをPlay Consoleにアップロードしてみましょう。警告が消えるだけでなく、App BundleのDownloadsタブを確認すると、AssetsにNative debug symbolsが追加されています。

![Google Play ConsoleのApp Bundle詳細画面でNative debug symbolsが追加された](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/e41a31f2-209d-43eb-a968-8b4fbcdafbc0.png)

## まとめ

- デバッグシンボルはクラッシュやANRの調査・分析に役に立つデータ
- `.aab` にデバッグシンボルを同梱するには、`app/build.gradle.kts` に `ndkVersion` と `ndk.debugSymbolLevel` を設定する

## 参考文献

- https://developer.android.com/topic/performance/app-optimization/enable-app-optimization#native-crash-support
- https://developer.android.com/ndk/downloads/
