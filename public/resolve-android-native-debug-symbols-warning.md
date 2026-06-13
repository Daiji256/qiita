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

# Google Play Consoleの「デバッグシンボルがありません」を解決する

## はじめに

Androidアプリをリリース・アップデートしようとした際、Google Play Console上で以下のワーニングが表示されたことはないでしょうか？

本記事では、Android App Bundle（`.aab` ファイル）にデバッグシンボルを含めるように設定し、この警告を解決する方法を紹介します。

> This App Bundle contains native code, and you've not uploaded debug symbols. We recommend you upload a symbol file to make your crashes and ANRs easier to analyze and debug.

> この App Bundle にはネイティブ コードが含まれ、デバッグ シンボルがアップロードされていません。クラッシュや ANR を簡単に分析、デバッグできるよう、シンボル ファイルをアップロードすることをおすすめします。

![TODO](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/7247fe64-8ede-45c0-abcb-e0403dd466c9.png)

## デバッグシンボルとは何か？

ソースコードをコンパイルしてバイナリ（マシン語）を作成する過程で、ファイルサイズを削減や最適化等により、変数名や関数名のようなソースコードの情報（デバッグ情報）が削除されます。

デバッグシンボルとは、このバイナリとソースコードを紐づけるためのマッピングデータです。

これがないと、そのバイナリ内での処理中にクラッシュが発生した際に、Play Console上からはどのようなエラーが起きたかの特定が困難になります。

## 必要な設定

以下のように `app/build.gradle.kts` に `ndkVersion` と `debugSymbolLevel` を設定することで、ビルドされる `.aab` ファイル内にデバッグシンボルが同梱されるようになります。

注意するべき点として、`ndkVersion` の明示的な指定が必要です。`debugSymbolLevel` だけを設定しても、ビルド時にデバッグシンボルが生成されても、`.aab` には含まれまれませんでした。

```kotlin
android {
    ndkVersion = "27.3.13750724" // 執筆時点での最新LTSバージョン

    buildTypes {
        release {
            ndk.debugSymbolLevel = "FULL"
        }
    }
}
```

## NDKをインストールする方法

以下の手順によりAndroid StudioでNDKをインストールできます。

1. メニューからTools > SDK Managerを選択する
2. SDK Toolsタブを開き、Show Package Detailsにチェックを入れる
3. インストールしたいNDKのバージョンにチェックを入れる
4. Applyする

![TODO](※ここにAndroid StudioのSDK Managerのスクリーンショットを挿入してください)

## 成功の確認方法

リリースビルドにて生成された `.aab` ファイルの拡張子を `.zip` に変更し解凍すると、 `BUNDLE-METADATA/com.android.tools.build.debugsymbols` が含まれているはずです。

実際にこの `.aab` ファイルをPlay Consoleにアップロードしてみましょう。警告が消えるだけでなく、App BundleのDownloadsタブを確認すると、AssetsにNative debug symbolsが追加されています。

![TODO](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/e41a31f2-209d-43eb-a968-8b4fbcdafbc0.png)

## まとめ

- デバッグシンボルはクラッシュやANRの調査・分析に役にたつデータ
- `.aab` にデバッグシンボルを同梱するには、`app/build.gradle.kts` に `ndkVersion` と `ndk.debugSymbolLevel = "FULL"` を設定する。

## 参考文献

- https://developer.android.com/topic/performance/app-optimization/enable-app-optimization#native-crash-support
- https://developer.android.com/ndk/downloads/
