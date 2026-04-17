# Microsoft Agent Framework による Agent 開発 (Python)

Azure AI Projects SDK (Python) を使って、プログラムからシンプルなエージェントおよびマルチエージェントワークフローを作成する方法を学びます。

---

## 1. 開発環境のセットアップ

### 1-1. 前提条件の確認

本ハンズオンでは以下が必要です。

- **Python 3.10 以上** がインストールされていること
- **Azure サブスクリプション** と **Microsoft Foundry リソース** が作成済みであること ([1. Azure ポータルから Microsoft Foundry リソースを作成](./1_basicagent.md#1-azure-ポータルから-microsoft-foundry-リソースを作成) 参照)
- **gpt-4.1** モデルがデプロイ済みであること

### 1-2. 必要なパッケージのインストール

ターミナル (またはコマンドプロンプト) を開き、以下のコマンドで必要な Python パッケージをインストールします。

```bash
pip install azure-ai-projects azure-identity
```

| パッケージ | 説明 |
|---|---|
| `azure-ai-projects` | Azure AI Foundry に接続してエージェントを管理する SDK |
| `azure-identity` | Azure への認証 (DefaultAzureCredential 等) を提供するパッケージ |

### 1-3. 接続文字列の確認

[Microsoft Foundry ポータル](https://ai.azure.com/nextgen) を開き、画面右上ツールバーの **ビルド** > 左メニューバーの **概要** をクリックして、プロジェクトの概要画面を表示します。

![](./images/3-1-01.png)

画面内にある **プロジェクト接続文字列** をコピーしておきます。接続文字列は次の形式になっています。

```
<HostName>;<AzureSubscriptionId>;<ResourceGroup>;<ProjectName>
```

![](./images/3-1-02.png)

> 接続文字列はプログラムから Foundry に接続する際に使用します。外部に公開しないよう注意してください。

---

## 2. シンプルなエージェントの作成

### 2-1. プロジェクトの作成

作業用のフォルダーを作成し、その中に Python スクリプトを作成します。

```bash
mkdir agent-framework-hol
cd agent-framework-hol
```

### 2-2. エージェントの作成と実行

`simple_agent.py` という名前のファイルを作成して、以下のコードを記述します。

`<接続文字列>` の部分を [1-3. 接続文字列の確認](#1-3-接続文字列の確認) でコピーしたプロジェクト接続文字列に置き換えます。

```python
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

# Foundry プロジェクトへの接続
project_client = AIProjectClient.from_connection_string(
    conn_str="<接続文字列>",
    credential=DefaultAzureCredential()
)

with project_client:
    # エージェントの作成
    agent = project_client.agents.create_agent(
        model="gpt-4.1",
        name="my-first-agent",
        instructions="あなたは親切な AI アシスタントです。ユーザーの質問に日本語で丁寧に回答してください。",
    )
    print(f"エージェントを作成しました: {agent.name} (ID: {agent.id})")

    # スレッド (会話) の作成
    thread = project_client.agents.create_thread()
    print(f"スレッドを作成しました: {thread.id}")

    # ユーザーメッセージの送信
    message = project_client.agents.create_message(
        thread_id=thread.id,
        role="user",
        content="Azure AI Foundry とは何ですか？簡単に教えてください。",
    )
    print(f"メッセージを送信しました: {message.id}")

    # エージェントの実行
    run = project_client.agents.create_and_process_run(
        thread_id=thread.id,
        agent_id=agent.id,
    )
    print(f"実行ステータス: {run.status}")

    # 応答メッセージの取得
    messages = project_client.agents.list_messages(thread_id=thread.id)
    for msg in messages:
        if msg.role == "assistant":
            print(f"\nエージェントの応答:\n{msg.content[0].text.value}")
            break

    # エージェントの削除 (クリーンアップ)
    project_client.agents.delete_agent(agent.id)
    print("\nエージェントを削除しました。")
```

### 2-3. Azure へのサインイン

スクリプトを実行する前に、Azure CLI でサインインしておきます。

```bash
az login
```

ブラウザが開き、Azure アカウントへのサインインを求められます。サインインが完了すると、ターミナルにサブスクリプションの一覧が表示されます。

![](./images/3-2-01.png)

### 2-4. スクリプトの実行

以下のコマンドでスクリプトを実行します。

```bash
python simple_agent.py
```

正常に実行されると、エージェントの作成・実行・応答取得・削除の各ステップが順に表示されます。

![](./images/3-2-02.png)

---

## 3. ツールを使ったエージェントの作成

### 3-1. コードインタープリターの追加

コードインタープリターを使ったエージェントを作成します。コードインタープリターを使うと、エージェントが Python コードを生成・実行して計算やデータ処理を行えるようになります。

`code_interpreter_agent.py` という名前のファイルを作成して、以下のコードを記述します。

```python
import os
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import CodeInterpreterTool
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient.from_connection_string(
    conn_str="<接続文字列>",
    credential=DefaultAzureCredential()
)

with project_client:
    # コードインタープリターツールの定義
    code_interpreter = CodeInterpreterTool()

    # ツールを持つエージェントの作成
    agent = project_client.agents.create_agent(
        model="gpt-4.1",
        name="code-interpreter-agent",
        instructions="あなたはデータ分析の専門家です。ユーザーの依頼に応じて Python コードを実行し、結果を分かりやすく説明してください。",
        tools=code_interpreter.definitions,
    )
    print(f"エージェントを作成しました: {agent.name} (ID: {agent.id})")

    # スレッドの作成とメッセージの送信
    thread = project_client.agents.create_thread()
    message = project_client.agents.create_message(
        thread_id=thread.id,
        role="user",
        content="1 から 100 までの整数の合計を計算してください。",
    )

    # エージェントの実行
    run = project_client.agents.create_and_process_run(
        thread_id=thread.id,
        agent_id=agent.id,
    )
    print(f"実行ステータス: {run.status}")

    # 応答メッセージの取得
    messages = project_client.agents.list_messages(thread_id=thread.id)
    for msg in messages:
        if msg.role == "assistant":
            for content_item in msg.content:
                if hasattr(content_item, "text"):
                    print(f"\nエージェントの応答:\n{content_item.text.value}")
            break

    # クリーンアップ
    project_client.agents.delete_agent(agent.id)
    print("\nエージェントを削除しました。")
```

スクリプトを実行します。

```bash
python code_interpreter_agent.py
```

コードインタープリターを使ったエージェントが Python コードを実行して、計算結果を返します。

![](./images/3-3-01.png)

### 3-2. ファイル検索ツールの追加

ファイル検索 (File Search) ツールを使うと、エージェントがアップロードされたファイルの内容を参照して回答できるようになります。

`file_search_agent.py` という名前のファイルを作成して、以下のコードを記述します。

> 事前に参照させたいテキストファイル (`sample.txt`) を同じフォルダーに用意してください。

```python
import os
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FileSearchTool
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient.from_connection_string(
    conn_str="<接続文字列>",
    credential=DefaultAzureCredential()
)

with project_client:
    # ファイルのアップロード
    with open("sample.txt", "rb") as f:
        uploaded_file = project_client.agents.upload_file_and_poll(
            file=f,
            purpose="assistants",
        )
    print(f"ファイルをアップロードしました: {uploaded_file.id}")

    # ベクターストアの作成とファイルの追加
    vector_store = project_client.agents.create_vector_store_and_poll(
        file_ids=[uploaded_file.id],
        name="my-vector-store",
    )
    print(f"ベクターストアを作成しました: {vector_store.id}")

    # ファイル検索ツールの定義
    file_search = FileSearchTool(vector_store_ids=[vector_store.id])

    # ツールを持つエージェントの作成
    agent = project_client.agents.create_agent(
        model="gpt-4.1",
        name="file-search-agent",
        instructions="あなたはアップロードされたファイルを参照して、ユーザーの質問に答えるアシスタントです。",
        tools=file_search.definitions,
        tool_resources=file_search.resources,
    )
    print(f"エージェントを作成しました: {agent.name} (ID: {agent.id})")

    # スレッドの作成とメッセージの送信
    thread = project_client.agents.create_thread()
    message = project_client.agents.create_message(
        thread_id=thread.id,
        role="user",
        content="アップロードされたファイルの内容を要約してください。",
    )

    # エージェントの実行
    run = project_client.agents.create_and_process_run(
        thread_id=thread.id,
        agent_id=agent.id,
    )
    print(f"実行ステータス: {run.status}")

    # 応答メッセージの取得
    messages = project_client.agents.list_messages(thread_id=thread.id)
    for msg in messages:
        if msg.role == "assistant":
            for content_item in msg.content:
                if hasattr(content_item, "text"):
                    print(f"\nエージェントの応答:\n{content_item.text.value}")
            break

    # クリーンアップ
    project_client.agents.delete_vector_store(vector_store.id)
    project_client.agents.delete_file(uploaded_file.id)
    project_client.agents.delete_agent(agent.id)
    print("\nクリーンアップが完了しました。")
```

スクリプトを実行します。

```bash
python file_search_agent.py
```

![](./images/3-3-02.png)

---

## 4. マルチエージェントワークフローの作成

複数のエージェントを連携させてタスクを処理するワークフローをプログラムで実装します。ここでは [2. Microsoft Foundry でマルチエージェントを作成する](./2_multiagent.md) と同等の処理を SDK で実現します。

- **トピック分類エージェント** (routing-agent): 入力内容を分類する
- **Microsoft 情報調査エージェント** (microsoft-agent): Microsoft 関連の情報を回答する
- **一般情報エージェント** (general-agent): その他のトピックを回答する

### 4-1. マルチエージェントワークフローの実装

`multi_agent_workflow.py` という名前のファイルを作成して、以下のコードを記述します。

```python
import json
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient.from_connection_string(
    conn_str="<接続文字列>",
    credential=DefaultAzureCredential()
)

def run_agent(project_client, agent_id, user_message):
    """指定したエージェントにメッセージを送り、応答テキストを返す"""
    thread = project_client.agents.create_thread()
    project_client.agents.create_message(
        thread_id=thread.id,
        role="user",
        content=user_message,
    )
    run = project_client.agents.create_and_process_run(
        thread_id=thread.id,
        agent_id=agent_id,
    )
    messages = project_client.agents.list_messages(thread_id=thread.id)
    for msg in messages:
        if msg.role == "assistant":
            for content_item in msg.content:
                if hasattr(content_item, "text"):
                    return content_item.text.value
    return ""


with project_client:
    # ── エージェントの作成 ──────────────────────────────────────

    # 1. トピック分類エージェント
    routing_agent = project_client.agents.create_agent(
        model="gpt-4.1",
        name="routing-agent",
        instructions=(
            "入力されたトピックを以下の選択肢に分類してください。"
            "選択肢の回答のみを返答してください。\n"
            "- Microsoft\n"
            "- Other"
        ),
    )
    print(f"routing-agent を作成しました (ID: {routing_agent.id})")

    # 2. Microsoft 情報調査エージェント
    microsoft_agent = project_client.agents.create_agent(
        model="gpt-4.1",
        name="microsoft-agent",
        instructions="マイクロソフトの製品やサービスについての情報を収集して、分かりやすく回答してください。",
    )
    print(f"microsoft-agent を作成しました (ID: {microsoft_agent.id})")

    # 3. 一般情報エージェント
    general_agent = project_client.agents.create_agent(
        model="gpt-4.1",
        name="general-agent",
        instructions="ユーザーの質問に対して、幅広いトピックについて分かりやすく回答してください。",
    )
    print(f"general-agent を作成しました (ID: {general_agent.id})")

    # ── ワークフローの実行 ──────────────────────────────────────

    user_input = "Azure AI Foundry の最新機能について教えてください。"
    print(f"\nユーザーの入力: {user_input}")

    # ステップ 1: トピック分類
    print("\n--- ステップ 1: トピック分類 ---")
    routing_result = run_agent(project_client, routing_agent.id, user_input)
    topic = routing_result.strip()
    print(f"分類結果: {topic}")

    # ステップ 2: 分類結果に応じてエージェントを選択
    print("\n--- ステップ 2: エージェントの選択と実行 ---")
    if "Microsoft" in topic:
        print("Microsoft 情報調査エージェントを使用します。")
        final_response = run_agent(project_client, microsoft_agent.id, user_input)
    else:
        print("一般情報エージェントを使用します。")
        final_response = run_agent(project_client, general_agent.id, user_input)

    print(f"\n最終応答:\n{final_response}")

    # ── クリーンアップ ──────────────────────────────────────────
    project_client.agents.delete_agent(routing_agent.id)
    project_client.agents.delete_agent(microsoft_agent.id)
    project_client.agents.delete_agent(general_agent.id)
    print("\n全エージェントを削除しました。")
```

### 4-2. ワークフローの実行

スクリプトを実行します。

```bash
python multi_agent_workflow.py
```

ワークフローが順に実行され、トピック分類の結果に応じて適切なエージェントが選択されて回答が生成されます。

![](./images/3-4-01.png)

> **動作の流れ**
> 1. ユーザーの入力を routing-agent が「Microsoft」または「Other」に分類します。
> 2. 分類結果に応じて、microsoft-agent または general-agent にユーザーの入力を渡します。
> 3. 選択されたエージェントが最終的な回答を生成します。

### 4-3. 複数ターンの会話への対応

同じスレッドを使い回すことで、エージェントとの複数ターンの会話を実現できます。以下のコードは、スレッドを保持しながら連続的にメッセージを送る例です。

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient.from_connection_string(
    conn_str="<接続文字列>",
    credential=DefaultAzureCredential()
)

with project_client:
    # エージェントの作成
    agent = project_client.agents.create_agent(
        model="gpt-4.1",
        name="multi-turn-agent",
        instructions="あなたは会話の文脈を理解する AI アシスタントです。前の会話を踏まえて回答してください。",
    )
    print(f"エージェントを作成しました: {agent.name}")

    # スレッドを 1 つ作成して会話全体で使い回す
    thread = project_client.agents.create_thread()
    print(f"スレッドを作成しました: {thread.id}\n")

    questions = [
        "Azure AI Foundry とは何ですか？",
        "それはどのようなユースケースに向いていますか？",
        "始めるにはどうすればいいですか？",
    ]

    for i, question in enumerate(questions, 1):
        print(f"[ターン {i}] ユーザー: {question}")
        project_client.agents.create_message(
            thread_id=thread.id,
            role="user",
            content=question,
        )
        run = project_client.agents.create_and_process_run(
            thread_id=thread.id,
            agent_id=agent.id,
        )
        messages = project_client.agents.list_messages(thread_id=thread.id)
        for msg in messages:
            if msg.role == "assistant":
                for content_item in msg.content:
                    if hasattr(content_item, "text"):
                        print(f"[ターン {i}] エージェント: {content_item.text.value}\n")
                break

    # クリーンアップ
    project_client.agents.delete_agent(agent.id)
    print("エージェントを削除しました。")
```

```bash
python multi_turn_agent.py
```

![](./images/3-4-02.png)

---

## 5. 既存エージェントの利用

Foundry ポータルで作成済みのエージェントを SDK から呼び出すこともできます。

### 5-1. 既存エージェントの ID を確認する

[Microsoft Foundry ポータル](https://ai.azure.com/nextgen) を開き、画面右上ツールバーの **ビルド** > 左メニューバーの **エージェント** から対象のエージェントを開きます。

プレイグラウンド画面の左上にある **エージェント ID** をコピーしておきます。

![](./images/3-5-01.png)

### 5-2. 既存エージェントを呼び出す

`use_existing_agent.py` という名前のファイルを作成して、以下のコードを記述します。`<エージェント ID>` の部分を 5-1 でコピーしたエージェント ID に置き換えます。

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient.from_connection_string(
    conn_str="<接続文字列>",
    credential=DefaultAzureCredential()
)

EXISTING_AGENT_ID = "<エージェント ID>"

with project_client:
    # 既存エージェントの取得
    agent = project_client.agents.get_agent(EXISTING_AGENT_ID)
    print(f"エージェントを取得しました: {agent.name} (ID: {agent.id})")

    # スレッドの作成とメッセージの送信
    thread = project_client.agents.create_thread()
    project_client.agents.create_message(
        thread_id=thread.id,
        role="user",
        content="こんにちは！あなたは何ができますか？",
    )

    # エージェントの実行
    run = project_client.agents.create_and_process_run(
        thread_id=thread.id,
        agent_id=agent.id,
    )
    print(f"実行ステータス: {run.status}")

    # 応答メッセージの取得
    messages = project_client.agents.list_messages(thread_id=thread.id)
    for msg in messages:
        if msg.role == "assistant":
            for content_item in msg.content:
                if hasattr(content_item, "text"):
                    print(f"\nエージェントの応答:\n{content_item.text.value}")
            break
```

スクリプトを実行します。

```bash
python use_existing_agent.py
```

![](./images/3-5-02.png)

> 既存エージェントを呼び出す場合、エージェント自体は削除しないよう注意してください。ポータルで作成・管理しているエージェントは、SDK からも参照・操作できます。

---

## まとめ

本ハンズオンでは、Azure AI Projects SDK を使って以下を学びました。

| 内容 | 概要 |
|---|---|
| シンプルなエージェントの作成 | `create_agent` / `create_thread` / `create_and_process_run` の基本的な使い方 |
| ツールの追加 | コードインタープリター・ファイル検索ツールをエージェントに追加する方法 |
| マルチエージェントワークフロー | 複数のエージェントを組み合わせてトピックに応じた処理を行う方法 |
| 複数ターンの会話 | スレッドを使い回して文脈を維持した会話を行う方法 |
| 既存エージェントの利用 | ポータルで作成したエージェントを SDK から呼び出す方法 |

SDK を使うことで、エージェントの作成・実行・管理をプログラムから柔軟に行えます。ポータル上での操作と組み合わせることで、より高度な AI アプリケーションを開発できます。

### 参考リンク

- [Azure AI Projects SDK (Python) ドキュメント](https://learn.microsoft.com/azure/ai-foundry/how-to/develop/sdk-overview)
- [Azure AI Agent Service クイックスタート](https://learn.microsoft.com/azure/ai-services/agents/quickstart)
- [azure-ai-projects PyPI](https://pypi.org/project/azure-ai-projects/)
