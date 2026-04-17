# Microsoft Agent Framework による Agent 開発

Microsoft Agent Framework SDK を使って、プログラムからエージェントを作成・実行する方法を学びます。

## ハンズオンコンテンツ一覧

| No. | ドキュメント | Notebook | 説明 |
|---|---|---|---|
| 3.1 | [Agent Framework による Agent 開発 (C#)](./3_AgentFramework/3.1_agentframework_csharp.md) | [C# Notebook](./3_AgentFramework/3.1_agentframework_csharp.ipynb) | Microsoft Agent Framework SDK (C#/.NET) を使って、プログラムからエージェントを作成・実行する方法を学びます。 |
| 3.2 | [Agent Framework による Agent 開発 (Python)](./3_AgentFramework/3.2_agentframework_python.md) | [Python Notebook](./3_AgentFramework/3.2_agentframework_python.ipynb) | Microsoft Agent Framework SDK (Python) を使って、プログラムからエージェントを作成・実行する方法を学びます。 |

サンプルの環境変数ファイルは [./3_AgentFramework/.env.sample](./3_AgentFramework/.env.sample) を参照してください。

## 前提条件

本ハンズオンでは以下が必要です。

- **Azure サブスクリプション** と **Microsoft Foundry リソース** が作成済みであること ([1. Azure ポータルから Microsoft Foundry リソースを作成](./1_basicagent.md) 参照)
- **gpt-4.1** モデルがデプロイ済みであること
- Foundry ポータルのプロジェクト接続文字列を取得済みであること

## 学習内容

どちらの言語版も以下の内容をカバーしています。

| 内容 | 概要 |
|---|---|
| 環境セットアップ | SDK のインストール、Foundry 接続文字列の取得 |
| シンプルなエージェントの作成 | エージェントの作成・実行・応答取得の基本フロー |
| ツールの追加 | コードインタープリター・ファイル検索ツールの組み込み方 |
| マルチエージェントワークフロー | 複数エージェントを連携してトピックに応じた処理を行う方法 |
| 複数ターンの会話 | スレッドを使い回して文脈を維持した会話を行う方法 |
| 既存エージェントの利用 | ポータルで作成済みのエージェントを SDK から呼び出す方法 |
