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

本記事は、[GitHub Actions Advent Calendar 2025](https://qiita.com/advent-calendar/2025/github-actions) の 18 日目の記事です。

## はじめに

GitHub Actions で生成されたファイルの保存する代表的な手段として、GitHub の artifacts 機能や S3 などの外部ストレージサービスがあります。

しかし、workflow で生成されたファイルを後続の workflow から参照したい、あるいは pull request の説明文から直接参照したいという場面が出てきます。これらの点で、artifacts 機能や外部ストレージサービスでは不便に感じることになります。

そこで本記事では orphan branch を利用してファイルを保存・取得するというアプローチと、それを GitHub Actions から利用する方法を紹介します。

## Orphan branch とは

Git では orphan branch という親を持たない空の branch を作成できます。通常の branch とは履歴が完全に分離されており、orphan branch を削除すれば、repository から完全に削除されるため、repository の肥大化を防ぐことができます。

この性質により、生成物専用の保存場所として利用できます。また、GitHub 上では通常のブランチと同様に扱えるため、raw.githubusercontent.com 経由でファイルを参照でき、pull request や issue の説明文などからファイルを直接参照できます。

## Orphan branch によるファイル管理の利点

GitHub の artifacts 機能は手軽で便利ですが、workflow 実行単位で管理され、保持期間がある点が用途によっては制約になります。また、pull request の説明文などからファイルを直接参照することができません。

外部のサービスを使う場合は、認証情報の管理や追加のインフラが必要になります。GitHub だけで完結させたい場合には、少し大げさに感じられるかもしれません。

これらと比べて orphan branch は、GitHub で完結し、workflow に依存せず,
保存期間を自分で管理でき、pull request の説明文などから参照できます。アプリ開発にて、UI のスクリーンショットテストの比較をするために、複数の workflow の結果を参照したかったり、pull request 上から直接スクリーンショットを確認したいということがありました。このときに、orphan branch を利用するようになりました。

この利点を活かし、ファイルの保存・取得・削除を GitHub Actions として共通化することにしました。

## 作成した GitHub Actions の紹介

ファイルの保存・取得・削除を orphan branch を用いて行うには、いくつかの基本的なコマンドと Git 操作ができれば十分です。

今回は、再利用性を高めるために、これらの操作を GitHub Actions としてまとめました。これによりより手軽に orphan branch を利用したファイル管理が可能になります。

詳しくはそれぞれの Actions の実装や[サンプル](https://github.com/Daiji256/orphan-branch-upload-download-delete-examples)を参考にしてください。

### [`daiji256/upload-to-orphan-branch`](https://github.com/Daiji256/upload-to-orphan-branch)

workflow 内で生成されたファイルを、指定した orphan branch にコミットして push する Action です。生成物を通常の開発履歴から分離したまま、repository 内に保存できます。

この例では `**/bar` を除く `dir` にあるファイルを `outputs-branch-name` という orphan branch に保存します。

```yaml
- uses: daiji256/upload-to-orphan-branch@v1
  with:
  branch: outputs-branch-name
  path: |
    dir
    !**/bar
```

### [`daiji256/download-from-orphan-branch`](https://github.com/Daiji256/download-from-orphan-branch)

指定した branch からファイルを取得し、ワークスペースに展開する Action です。別の workflow で事前に保存したファイルを再利用できます。

この例では `outputs-branch-name` という branch からファイルを取得し、`dir` に展開します。

```yaml
- uses: daiji256/download-from-orphan-branch@v1
  with:
  branch: outputs-branch-name
  path: dir
```

### [`daiji256/delete-orphan-branch`](https://github.com/Daiji256/delete-orphan-branch)

不要になった orphan branch を削除するための Action です。生成物ごとに branch を作成する運用では、定期的なクリーンアップが重要になります。

この例では、`orphan-output-` で始まる branch のうち、7 日以上前に作成されたものを削除します。

```yaml
- uses: daiji256/delete-orphan-branch@v1
  with:
    branch-regex: orphan-output-.*
    older-than-seconds: 604800 # 7 days
```

## 現状の課題と今後の改善

現状の実装にはいくつか課題があります。Linux 環境での動作確認に留まっている点や、git コマンド失敗時などのエラーハンドリングが十分とは言えません。

今後は、TypeScript での再実装や、macOS・Windows を含めた環境での動作確認、エラーハンドリングとログ出力の強化を進めたいと考えています。

## おわりに

orphan branch を使ったファイル管理は、すべてのケースに適した方法ではありませんが、GitHub Actions の中だけで生成物を扱い、後から参照できる形で残したい場合には有効な選択肢になります。

ぜひ、orphan branch を活用したファイル管理を試してみてください。また、今回紹介した GitHub Actions を利用してもらえると嬉しいです。

## 参考文献

- [Git - git-switch Documentation](https://git-scm.com/docs/git-switch)
- [GitHub Actions documentation - GitHub Docs](https://docs.github.com/en/actions)
- [daiji256/upload-to-orphan-branch](https://github.com/Daiji256/upload-to-orphan-branch)
- [daiji256/download-from-orphan-branch](https://github.com/Daiji256/download-from-orphan-branch)
- [daiji256/delete-orphan-branch](https://github.com/Daiji256/delete-orphan-branch)
- [Orphan Branch Upload / Download / Delete Examples](https://github.com/Daiji256/orphan-branch-upload-download-delete-examples)
- [Daiji256/android-showcase/.github/workflows/unit-test.yml](https://github.com/Daiji256/android-showcase/blob/1010840b0eb7873f50a405d6199e904ae2f0a28d/.github/workflows/unit-test.yml)
