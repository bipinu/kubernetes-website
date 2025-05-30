---
layout: blog
title: "KubernetesのマルチコンテナPod: 概要"
date: 2025-04-22
slug: multi-container-pods-overview
author: Agata Skorupka (The Scale Factory)
translator: >
  [Takuya Kitamura](https://github.com/kfess)
---

クラウドネイティブアーキテクチャの進化が続く中、Kubernetesは複雑で分散したシステムをデプロイするための定番のプラットフォームとなってきました。
このエコシステムにおける最も強力でありながら繊細な設計パターンの一つがサイドカーパターンです。これは、開発者がソースコードに深く踏み込むことなく、アプリケーションの機能を拡張できる手法です。

## サイドカーパターンの起源

サイドカーは、バイクに取り付ける信頼できる補助座席のようなものだと考えてみてください。
ITインフラストラクチャでは、重要な処理を担うために、補助的なサービスが従来から利用されてきました。
コンテナが登場する以前は、ロギング、モニタリング、ネットワーク処理を管理するために、バックグラウンドプロセスやヘルパーデーモンに依存していました。
マイクロサービスの革命により、このアプローチは変革され、サイドカーは体系的かつ意図的なアーキテクチャの選択肢となりました。
マイクロサービスの台頭に伴い、サイドカーパターンはより明確に定義されるようになり、開発者はメインサービスのコードを変更することなく、特定の責務を切り離せるようになりました。
IstioやLinkerdのようなサービスメッシュは、サイドカープロキシを普及させ、これらの補助的なコンテナが分散システムにおける可観測性、セキュリティ、トラフィック管理を洗練された方法で処理できることを示しました。

## Kubernetesにおける実装

Kubernetesでは、[サイドカーコンテナ](/ja/docs/concepts/workloads/pods/sidecar-containers/)はメインのアプリケーションと同じPod内で動作し、通信やリソースの共有を可能にします。
これは、単にPod内に複数のコンテナを並列に定義することのように聞こえるかもしれません。
実際、その通りであり、Kubernetes v1.29.0でサイドカーのネイティブサポートが導入されるまでは、そのように実装する必要がありました。
現在では、Podマニフェスト内で`spec.initContainers`フィールドを使用してサイドカーコンテナを定義することができます。
これをサイドカーコンテナとして機能させるポイントは、`restartPolicy: Always`を指定することです。
以下はその一例で、Kubernetesマニフェスト全体の一部を抜粋したものです。

```yaml
initContainers:
  - name: logshipper
    image: alpine:latest
    restartPolicy: Always
  command: ['sh', '-c', 'tail -F /opt/logs.txt']
    volumeMounts:
    - name: data
        mountPath: /opt
```

`spec.initContainers`というフィールド名は、混乱を招くかもしれません。
サイドカーコンテナを定義したいのに、なぜ`spec.initContainers`配列にエントリを追加しなければならないのでしょうか？
`spec.initContainers`に定義されたコンテナは、メインアプリケーションが起動する直前に一度だけ実行され、完了すると終了します。
一方、サイドカーコンテナは通常、メインのアプリケーションコンテナと並行して動作し続けます。
Kubernetesにおけるネイティブなサイドカーコンテナは、`spec.initContainers`に`restartPolicy:Always`を指定することで、従来の[Initコンテナ](/ja/docs/concepts/workloads/pods/init-containers/)とは異なる挙動を持ち、常に稼働し続けることが保証されます。

## サイドカーを採用すべき場合と避けるべき場合

サイドカーパターンは多くのケースで有用ですが、正当化されるようなユースケースがない限り、一般的には推奨される手法ではありません。
サイドカーを追加すると、複雑性やリソース消費、ネットワーク遅延の可能性が増大します。
その代わりに、まずは組み込みライブラリや共通インフラなど、より単純な代替手段を検討すべきです。

**サイドカーの導入が適しているのは次のような場合です:**

1. 元のコードに手を加えることなくアプリケーションの機能を拡張する必要がある場合
1. ロギング、モニタリング、セキュリティなどの横断的な考慮が必要な実装をする場合
1. モダンなネットワーク機能を必要とするレガシーアプリケーションを扱う場合
1. 独立したスケーリングや更新が求められるマイクロサービスを設計する場合

**次のような場合は慎重に検討してください:**

1. リソース効率を最優先したい場合
1. 最小限のネットワーク遅延が重要な場合
1. より単純な代替手段が存在する場合
1. トラブルシューティングの複雑さを最小限に抑えたい場合

## 4つの重要なマルチコンテナパターン

### Initコンテナパターン

**Initコンテナ**パターンは、メインのアプリケーションコンテナが起動する前に(しばしば重要な)初期化処理を実行するために使用されます。
通常のコンテナと異なり、Initコンテナは処理が完了すると終了し、メインアプリケーションの前提条件が満たされることを保証します。

**このパターンが適しているケース:**

1. 各種設定の準備
1. シークレットの読み込み
1. 依存関係の利用可能性の確認
1. データベースマイグレーションの実行

Initコンテナを使用することで、アプリケーションのコードを変更することなく、予測可能で制御された環境下での起動を実現できます。

### Ambassadorパターン

Ambassadorコンテナは、Pod内で動作する補助的なサービスを提供し、ネットワークサービスへのアクセスを簡易化します。
一般的に、Ambassadorコンテナはアプリケーションコンテナに代わってネットワークリクエストを送信し、サービス検出、ピアの識別検証、通信の暗号化といった処理を担います。

**このパターンが特に有効なのは次のような場合です:**

1. クライアント接続に関する処理を切り離す場合
1. 言語に依存しないネットワーク機能を実装する場合
1. TLSなどのセキュリティ層を追加する場合
1. 堅牢なサーキットブレーカーやリトライ機構を構築する場合

### Configuration helper

_configuration helper_ サイドカーは、アプリケーションに対して設定の更新を動的に提供し、サービスを中断させることなく常に最新の設定にアクセスできるようにします。
多くの場合、アプリケーションが正常に起動するためには、事前に初期設定を提供する必要があります。

**ユースケース:**

1. 環境変数やシークレットの取得
1. 設定変更のポーリング
1. 設定管理とアプリケーションロジックの分離

### Adapterパターン

_adapter_(または _façade_)コンテナは、メインのアプリケーションコンテナと外部サービスとの間の相互運用性を実現します。
これは、データ形式、プロトコル、またはAPIの変換を行うことで実現されます。

**このパターンの強み:**

1. レガシーなデータ形式の変換
1. 通信プロトコル間の橋渡し
1. 互換性のないサービス間の統合促進

## まとめ

サイドカーパターンは非常に高い柔軟性を提供してくれますが、銀の弾丸ではありません。
サイドカーを追加するたびに、複雑性が増し、リソースを消費し、運用負荷が高まる可能性があります。
まずは、より単純な代替手段を検討するようにしてください。
鍵となるのは、戦略的な実装です。
サイドカーは、あらゆる場面で使うデフォルトの方法ではなく、特定のアーキテクチャ上の課題を解決するための精密なツールとして活用すべきです。
適切に使用すれば、コンテナ化された環境において、セキュリティ、ネットワーキング、設定管理の向上に貢献できます。
賢明に選び、注意深く実装し、サイドカーを活用してコンテナエコシステムをさらに高めましょう。
