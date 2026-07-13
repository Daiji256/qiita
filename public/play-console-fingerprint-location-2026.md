---
title: Play Consoleでアプリ署名のフィンガープリントを確認する（2026年）
tags:
  - Android
  - fingerprint
  - keystore
  - 署名
  - GooglePlayConsole
private: false
updated_at: '2026-07-04T08:40:09+09:00'
id: dfcbc6972c31558657e7
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

Androidアプリを開発していると、Google MapsやApp Linksなどの外部サービス連携のために、アプリ署名のフィンガープリント（SHA-1やSHA-256）を確認したい場面があります。フィンガープリントは、インストール済みのアプリ（APK）もしくは手元のキーストア（Keystore）からコマンド等を用いて確認する方法が一般的です。

しかし、現在のAndroidアプリ開発ではPlayアプリ署名（Play App Signing）を利用してGoogle Play側で署名鍵を管理することが主流となっており、手元にリリース用のキーストアを持たないことも多いでしょう。そのような場合、正確なフィンガープリントはGoogle Play Console上で確認する必要があります。

本記事では、2026年7月におけるPlay Console上でのフィンガープリント確認手順を紹介します。

## Play Console上でアプリ署名のフィンガープリントを確認する方法

最近の[Play Console](https://play.google.com/console)のUI変更により、アプリ署名のフィンガープリント（SHA-1 / SHA-256）を確認する場所が変わりました。

2026年6月19日現在は、以下の手順でフィンガープリントを確認できます：

1. サイドメニューから **Google Play による保護（Protected with Play）** を選択する
2. **Google Play ストアの保護（Play Store protection）** の項目を展開する
3. **アプリ署名鍵の保護（Protect app signing key）** の右側にある **Play アプリ署名の管理（Manage Play app signing）** ボタンをクリックする
4. **遷移先の アプリの署名（App signing）** ページで各種フィンガープリントを確認できる

![Google Play による保護（Protected with Play）にてGoogle Playストアの保護（Play Store protection）が展開されている](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/2751248c-0842-4fca-9799-e687d7af4518.png)

![アプリの署名（App signing）にてフィンガープリントが表示されている](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/e01bf3f9-ede4-4d72-aace-599e9095c092.png)
