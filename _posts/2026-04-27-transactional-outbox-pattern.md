---
title: Transactional Outbox Patternとは
tags: AWS マイクロサービス 設計パターン
excerpt: マイクロサービスにおけるデュアルライト問題を解決するAWSサンプルを読んだ。
---

マイクロサービスでDBへの書き込みとイベント通知を同時に行う際、どちらか一方が失敗するとシステムが不整合になる。Transactional Outbox PatternはこのデュアルライトをOutboxテーブルやCDC（Change Data Capture）で解決するアーキテクチャだ。AWSサンプルではAurora＋SQSとDynamoDB＋Kinesisの2つの実装が提供されている。

参考：[aws-samples/transactional-outbox-pattern](https://github.com/aws-samples/transactional-outbox-pattern)
