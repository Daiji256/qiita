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

# 【Android】Play Consoleの「ネイティブデバッグシンボルがありません」警告をAABビルドで自動解決する正攻法

## はじめに

Androidアプリ（特にC/C++のネイティブコードを含むアプリ）をGoogle Play Consoleにアップロードした際、以下のようなワーニングに遭遇したことはないでしょうか？

> **Warning**
> This App Bundle contains native code, and you've not uploaded debug symbols. We recommend you upload a symbol file to make your crashes and ANRs easier to analyze and debug.

![Play Consoleの警告画像]
*(※ここに1枚目の画像「Screenshot 2026-05-23 at 5.02.04 PM.png」を挿入)*

この警告を消すために、ビルド途中で生成される中間ファイル（`app/build/intermediates/merged_native_libs/.../out/lib`）を手動でzip圧縮してアップロードするというワークアラウンドがよく知られています。しかし、毎回手動でzipを作るのは面倒ですし、CI/CDに乗せるのもスマートではありません。

本来、**Android App Bundle (AAB) にはビルド時にネイティブデバッグシンボルを自動的に同梱する機能**が備わっています。

本記事では、Gradle Kotlin DSL (`build.gradle.kts`) と Version Catalog (`libs.versions.toml`) を使って、この問題を正攻法で（手動zipアップロードなしで）解決する方法を解説します。

---

## 結論：2つの設定が必須

「AABにシンボルを含めるには `debugSymbolLevel` を設定すればよい」という情報は見かけますが、それだけでは不十分（サイレントにスキップされる）なケースが多々あります。

以下の2つをセットで設定することが必須です。

1. **`ndkVersion` の明示的な指定**
2. **`debugSymbolLevel = "FULL"` の指定**

---

## 具体的な設定手順

最近のAndroidプロジェクトの標準である Version Catalog を使った設定例を紹介します。

### 1. `libs.versions.toml` にNDKバージョンを定義する

まずは、使用するNDKのバージョンを定義します。バージョン番号は、Android Studioの `SDK Manager` > `SDK Tools` > `NDK (Side by side)` （※Show Package Detailsにチェック）から確認できます。

### 2. `build.gradle.kts` に設定を追記する

アプリケーショレベルの `build.gradle.kts` を開き、`android` ブロック内に設定を追加します。

---

## 罠の正体：なぜ `ndkVersion` が必要なのか？

`debugSymbolLevel` を設定したのにAABにシンボルが含まれない（警告が消えない）原因の多くは、**`ndkVersion` の未指定**にあります。

Android Gradle Plugin (AGP) は、`.so` ファイルからデバッグシンボルを抽出する際、NDKに含まれるツール（`llvm-objcopy` など）を使用します。
しかし `ndkVersion` が明示されていない場合、AGPがツールのパスを正しく解決できず、**エラーを吐くことなく抽出タスク（`extractReleaseNativeDebugMetadata`）をサイレントにスキップしてしまう**ことがあるのです。

そのため、「ビルドは通るし設定も書いたのに、AABにシンボルが含まれない」という現象に陥りやすくなります。

---

## 成功の確認方法

上記の設定を行ってリリースビルド（`./gradlew bundleRelease` など）を実行し、生成された `.aab` ファイルを Play Console にアップロードしてみましょう。

警告が消えるだけでなく、App Bundle エクスプローラーの「ダウンロード（Downloads）」タブを確認すると、**「Native debug symbols (native-debug-symbols.zip)」** という項目が自動的に追加されているはずです。

![Play Consoleの成功画像]
*(※ここに2枚目の画像「Screenshot 2026-05-23 at 17-04-05...png」を挿入)*

これで、クラッシュ時のスタックトレース（C/C++レイヤー）がPlay Console上で適切に難読化解除（デコード）されるようになります。

## まとめ

* 警告を消すために中間ファイルをわざわざ手動でzipにする必要はない。
* AABに自動でシンボルを同梱するには `ndk.debugSymbolLevel = "FULL"` を設定する。
* **ただし、`ndkVersion` を明示しないと無視されることがあるので必ずセットで設定する。**

ネイティブコードを含むAndroidアプリを開発していて、Play Consoleの警告に悩まされている方の参考になれば幸いです！
