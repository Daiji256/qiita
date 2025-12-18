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

# GitHub Actions で orphan branch を使ってファイルを保存・取得する方法

## はじめに

GitHub Actions を使って CI やテストを回していると、ワークフローの中で生成されたファイルを後続のワークフローから参照したい、あるいは pull request 上から直接確認できる形で残しておきたい、という場面があります。

代表的な手段としては GitHub Actions の Artifacts 機能や、S3 などの外部ストレージサービスがあります。一方で、実際に使ってみると、保持期間や参照方法、認証情報の管理といった点が気になるケースもありました。

そこで本記事では、GitHub の orphan branch を利用してファイルを保存・取得するというアプローチと、それを GitHub Actions として実装した例を紹介します。GitHub Actions Advent Calendar 2025 の一記事として、実運用の中で得られた知見を整理します。

---

## orphan branch とは

orphan branch は、親となるコミットを持たないブランチです。通常のブランチとは履歴が完全に分離されており、リポジトリのメインの履歴には影響を与えません。

この性質により、生成物専用の保存場所として利用できます。また、GitHub 上では通常のブランチと同様に扱えるため、raw.githubusercontent.com 経由でファイルを参照でき、pull request や Issue に URL を貼ることも可能です。

---

## orphan branch を使おうと考えた背景

Artifacts 機能は手軽で便利ですが、ワークフロー実行単位で管理され、保持期間がある点が用途によっては制約になります。また、pull request の説明文などから直接参照するにはやや工夫が必要です。

外部ストレージサービスを使う場合は、認証情報の管理や追加のインフラが必要になります。CI の中だけで完結させたい場合には、少し大げさに感じることもあります。

これらと比べて orphan branch は、GitHub の権限だけで完結し、保存期間を特に意識せずにファイルを残せる点が魅力でした。この利点を活かし、ファイルの保存・取得・整理を GitHub Actions として共通化することにしました。

---

## 作成した GitHub Actions の概要

今回作成したのは、orphan branch を使ったファイル管理を行うための 3 つの GitHub Actions と、それらをまとめたサンプルリポジトリです。

### upload-to-orphan-branch

ワークフロー内で生成されたファイルを、指定した orphan branch にコミットして push する Action です。生成物を通常の開発履歴から分離したまま、リポジトリ内に保存できます。

### download-from-orphan-branch

指定したブランチからファイルを取得し、ワークスペースに展開する Action です。別ジョブや別ワークフローから、事前に保存した生成物を再利用できます。

### delete-orphan-branch

不要になった orphan branch を削除するための Action です。生成物ごとにブランチを作成する運用では、定期的なクリーンアップが重要になります。

### サンプルリポジトリ

これら 3 つの Action を組み合わせた利用例として、アップロードからダウンロード、削除までの一連の流れを確認できるサンプルリポジトリも用意しています。

---

## 使い方の例

### ファイルを保存する

```yaml
- name: Upload generated file
  uses: Daiji256/upload-to-orphan-branch@v1
  with:
    branch: generated-files/example
    path: output/result.json
```

この例では、`output/result.json` を `generated-files/example` という orphan branch に保存します。保存されたファイルは、GitHub 上から URL として参照できます。

### 保存したファイルを取得する

```yaml
- name: Download generated file
  uses: Daiji256/download-from-orphan-branch@v1
  with:
    branch: generated-files/example
    path: output
```

別のワークフローからでも、同じブランチを指定することでファイルを取得できます。

### 不要になったブランチを削除する

```yaml
- name: Cleanup orphan branches
  uses: Daiji256/delete-orphan-branch@v1
  with:
    branch-pattern: generated-files/.*
```

定期実行と組み合わせることで、リポジトリ内の orphan branch が増えすぎるのを防げます。

---

## 実際の利用例

筆者はアプリ開発の文脈で、UI テスト時に生成したスクリーンショットの保存や、過去との差分比較、その結果を pull request 上から参照する目的でこの仕組みを利用しています。具体的なワークフロー例は、以下の設定に含まれています。

* [https://github.com/Daiji256/android-showcase/blob/1010840b0eb7873f50a405d6199e904ae2f0a28d/.github/workflows/unit-test.yml](https://github.com/Daiji256/android-showcase/blob/1010840b0eb7873f50a405d6199e904ae2f0a28d/.github/workflows/unit-test.yml)

---

## 現状の課題と今後の改善

現状の実装にはいくつか課題があります。Linux 環境での動作確認に留まっている点や、git コマンド失敗時などのエラーハンドリングが十分とは言えません。

今後は、TypeScript での再実装や、macOS・Windows を含めた環境での動作確認、エラーハンドリングとログ出力の強化を進めたいと考えています。

---

## おわりに

orphan branch を使ったファイル管理は、すべてのケースに適した方法ではありませんが、GitHub Actions の中だけで生成物を扱い、後から参照できる形で残したい場合には有効な選択肢になります。

Artifacts や外部ストレージと使い分ける前提で、一つの実装例として参考になれば幸いです。

---

## 参考文献

* GitHub Docs: Branches
* GitHub Docs: GitHub Actions
* upload-to-orphan-branch リポジトリ
* download-from-orphan-branch リポジトリ
* delete-orphan-branch リポジトリ
