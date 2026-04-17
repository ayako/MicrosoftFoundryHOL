# Microsoft Agent Framework による Agent 開発 (C#)

Azure AI Projects SDK (C#/.NET) を使って、プログラムからシンプルなエージェントおよびマルチエージェントワークフローを作成する方法を学びます。

---

## 1. 開発環境のセットアップ

### 1-1. 前提条件の確認

本ハンズオンでは以下が必要です。

- **.NET 8.0 以上** がインストールされていること
- **Azure サブスクリプション** と **Microsoft Foundry リソース** が作成済みであること ([1. Azure ポータルから Microsoft Foundry リソースを作成](./1_basicagent.md#1-azure-ポータルから-microsoft-foundry-リソースを作成) 参照)
- **gpt-4.1** モデルがデプロイ済みであること

### 1-2. プロジェクトの作成

ターミナル (またはコマンドプロンプト) を開き、以下のコマンドで新しい C# コンソールアプリを作成します。

```bash
dotnet new console -n AgentFrameworkHOL
cd AgentFrameworkHOL
```

必要な NuGet パッケージをインストールします。

```bash
dotnet add package Azure.AI.Projects --prerelease
dotnet add package Azure.Identity
```

| パッケージ | 説明 |
|---|---|
| `Azure.AI.Projects` | Azure AI Foundry に接続してエージェントを管理する SDK |
| `Azure.Identity` | Azure への認証 (DefaultAzureCredential 等) を提供するパッケージ |

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

### 2-1. エージェントの作成と実行

`Program.cs` を開き、以下のコードに書き換えます。

`<接続文字列>` の部分を [1-3. 接続文字列の確認](#1-3-接続文字列の確認) でコピーしたプロジェクト接続文字列に置き換えます。

```csharp
using Azure.AI.Projects;
using Azure.Identity;

// Foundry プロジェクトへの接続
var connectionString = "<接続文字列>";
var projectClient = new AIProjectClient(connectionString, new DefaultAzureCredential());
var agentsClient = projectClient.GetAgentsClient();

// エージェントの作成
var agent = await agentsClient.CreateAgentAsync(
    model: "gpt-4.1",
    name: "my-first-agent",
    instructions: "あなたは親切な AI アシスタントです。ユーザーの質問に日本語で丁寧に回答してください。"
);
Console.WriteLine($"エージェントを作成しました: {agent.Value.Name} (ID: {agent.Value.Id})");

// スレッド (会話) の作成
var thread = await agentsClient.CreateThreadAsync();
Console.WriteLine($"スレッドを作成しました: {thread.Value.Id}");

// ユーザーメッセージの送信
var message = await agentsClient.CreateMessageAsync(
    threadId: thread.Value.Id,
    role: MessageRole.User,
    content: "Azure AI Foundry とは何ですか？簡単に教えてください。"
);
Console.WriteLine($"メッセージを送信しました: {message.Value.Id}");

// エージェントの実行
var run = await agentsClient.CreateRunAsync(thread.Value.Id, agent.Value.Id);

// 実行完了まで待機
do
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    run = await agentsClient.GetRunAsync(thread.Value.Id, run.Value.Id);
} while (run.Value.Status == RunStatus.Queued || run.Value.Status == RunStatus.InProgress);

Console.WriteLine($"実行ステータス: {run.Value.Status}");

// 応答メッセージの取得
var messages = agentsClient.GetMessagesAsync(thread.Value.Id);
await foreach (var msg in messages)
{
    if (msg.Role == MessageRole.Agent)
    {
        foreach (var contentItem in msg.ContentItems)
        {
            if (contentItem is MessageTextContent textContent)
            {
                Console.WriteLine($"\nエージェントの応答:\n{textContent.Text}");
            }
        }
        break;
    }
}

// エージェントの削除 (クリーンアップ)
await agentsClient.DeleteAgentAsync(agent.Value.Id);
Console.WriteLine("\nエージェントを削除しました。");
```

### 2-2. Azure へのサインイン

アプリを実行する前に、Azure CLI でサインインしておきます。

```bash
az login
```

ブラウザが開き、Azure アカウントへのサインインを求められます。

![](./images/3-2-01.png)

### 2-3. アプリの実行

以下のコマンドでアプリを実行します。

```bash
dotnet run
```

正常に実行されると、エージェントの作成・実行・応答取得・削除の各ステップが順に表示されます。

![](./images/3-2-02.png)

---

## 3. ツールを使ったエージェントの作成

### 3-1. コードインタープリターの追加

コードインタープリターを使ったエージェントを作成します。コードインタープリターを使うと、エージェントが Python コードを生成・実行して計算やデータ処理を行えるようになります。

`Program.cs` の内容を以下に書き換えます。

```csharp
using Azure.AI.Projects;
using Azure.Identity;

var connectionString = "<接続文字列>";
var projectClient = new AIProjectClient(connectionString, new DefaultAzureCredential());
var agentsClient = projectClient.GetAgentsClient();

// コードインタープリターツールの定義
var codeInterpreterTool = new CodeInterpreterToolDefinition();

// ツールを持つエージェントの作成
var agent = await agentsClient.CreateAgentAsync(
    model: "gpt-4.1",
    name: "code-interpreter-agent",
    instructions: "あなたはデータ分析の専門家です。ユーザーの依頼に応じて Python コードを実行し、結果を分かりやすく説明してください。",
    tools: [codeInterpreterTool]
);
Console.WriteLine($"エージェントを作成しました: {agent.Value.Name} (ID: {agent.Value.Id})");

// スレッドの作成とメッセージの送信
var thread = await agentsClient.CreateThreadAsync();
await agentsClient.CreateMessageAsync(
    threadId: thread.Value.Id,
    role: MessageRole.User,
    content: "1 から 100 までの整数の合計を計算してください。"
);

// エージェントの実行
var run = await agentsClient.CreateRunAsync(thread.Value.Id, agent.Value.Id);
do
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    run = await agentsClient.GetRunAsync(thread.Value.Id, run.Value.Id);
} while (run.Value.Status == RunStatus.Queued || run.Value.Status == RunStatus.InProgress);

Console.WriteLine($"実行ステータス: {run.Value.Status}");

// 応答の取得
var messages = agentsClient.GetMessagesAsync(thread.Value.Id);
await foreach (var msg in messages)
{
    if (msg.Role == MessageRole.Agent)
    {
        foreach (var contentItem in msg.ContentItems)
        {
            if (contentItem is MessageTextContent textContent)
            {
                Console.WriteLine($"\nエージェントの応答:\n{textContent.Text}");
            }
        }
        break;
    }
}

// クリーンアップ
await agentsClient.DeleteAgentAsync(agent.Value.Id);
Console.WriteLine("\nエージェントを削除しました。");
```

```bash
dotnet run
```

コードインタープリターを使ったエージェントが Python コードを実行して、計算結果を返します。

![](./images/3-3-01.png)

### 3-2. ファイル検索ツールの追加

ファイル検索 (File Search) ツールを使うと、エージェントがアップロードされたファイルの内容を参照して回答できるようになります。

> 事前に参照させたいテキストファイル (`sample.txt`) を同じフォルダーに用意してください。

`Program.cs` の内容を以下に書き換えます。

```csharp
using Azure.AI.Projects;
using Azure.Identity;

var connectionString = "<接続文字列>";
var projectClient = new AIProjectClient(connectionString, new DefaultAzureCredential());
var agentsClient = projectClient.GetAgentsClient();

// ファイルのアップロード
using var fileStream = File.OpenRead("sample.txt");
var uploadedFile = await agentsClient.UploadFileAsync(fileStream, AgentFilePurpose.Agents, "sample.txt");
Console.WriteLine($"ファイルをアップロードしました: {uploadedFile.Value.Id}");

// ベクターストアの作成とファイルの追加
var vectorStore = await agentsClient.CreateVectorStoreAsync(
    fileIds: [uploadedFile.Value.Id],
    name: "my-vector-store"
);
Console.WriteLine($"ベクターストアを作成しました: {vectorStore.Value.Id}");

// ファイル検索ツールの定義
var fileSearchTool = new FileSearchToolDefinition();
var toolResources = new ToolResources
{
    FileSearch = new FileSearchToolResource([vectorStore.Value.Id])
};

// ツールを持つエージェントの作成
var agent = await agentsClient.CreateAgentAsync(
    model: "gpt-4.1",
    name: "file-search-agent",
    instructions: "あなたはアップロードされたファイルを参照して、ユーザーの質問に答えるアシスタントです。",
    tools: [fileSearchTool],
    toolResources: toolResources
);
Console.WriteLine($"エージェントを作成しました: {agent.Value.Name} (ID: {agent.Value.Id})");

// スレッドの作成とメッセージの送信
var thread = await agentsClient.CreateThreadAsync();
await agentsClient.CreateMessageAsync(
    threadId: thread.Value.Id,
    role: MessageRole.User,
    content: "アップロードされたファイルの内容を要約してください。"
);

// エージェントの実行
var run = await agentsClient.CreateRunAsync(thread.Value.Id, agent.Value.Id);
do
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    run = await agentsClient.GetRunAsync(thread.Value.Id, run.Value.Id);
} while (run.Value.Status == RunStatus.Queued || run.Value.Status == RunStatus.InProgress);

Console.WriteLine($"実行ステータス: {run.Value.Status}");

// 応答の取得
var messages = agentsClient.GetMessagesAsync(thread.Value.Id);
await foreach (var msg in messages)
{
    if (msg.Role == MessageRole.Agent)
    {
        foreach (var contentItem in msg.ContentItems)
        {
            if (contentItem is MessageTextContent textContent)
            {
                Console.WriteLine($"\nエージェントの応答:\n{textContent.Text}");
            }
        }
        break;
    }
}

// クリーンアップ
await agentsClient.DeleteVectorStoreAsync(vectorStore.Value.Id);
await agentsClient.DeleteFileAsync(uploadedFile.Value.Id);
await agentsClient.DeleteAgentAsync(agent.Value.Id);
Console.WriteLine("\nクリーンアップが完了しました。");
```

```bash
dotnet run
```

![](./images/3-3-02.png)

---

## 4. マルチエージェントワークフローの作成

複数のエージェントを連携させてタスクを処理するワークフローをプログラムで実装します。ここでは [2. Microsoft Foundry でマルチエージェントを作成する](./2_multiagent.md) と同等の処理を SDK で実現します。

- **トピック分類エージェント** (routing-agent): 入力内容を分類する
- **Microsoft 情報調査エージェント** (microsoft-agent): Microsoft 関連の情報を回答する
- **一般情報エージェント** (general-agent): その他のトピックを回答する

### 4-1. マルチエージェントワークフローの実装

`Program.cs` の内容を以下に書き換えます。

```csharp
using Azure.AI.Projects;
using Azure.Identity;

var connectionString = "<接続文字列>";
var projectClient = new AIProjectClient(connectionString, new DefaultAzureCredential());
var agentsClient = projectClient.GetAgentsClient();

// 指定したエージェントにメッセージを送り、応答テキストを返すヘルパー
async Task<string> RunAgentAsync(AgentsClient client, string agentId, string userMessage)
{
    var thread = await client.CreateThreadAsync();
    await client.CreateMessageAsync(thread.Value.Id, MessageRole.User, userMessage);

    var run = await client.CreateRunAsync(thread.Value.Id, agentId);
    do
    {
        await Task.Delay(TimeSpan.FromSeconds(1));
        run = await client.GetRunAsync(thread.Value.Id, run.Value.Id);
    } while (run.Value.Status == RunStatus.Queued || run.Value.Status == RunStatus.InProgress);

    var messages = client.GetMessagesAsync(thread.Value.Id);
    await foreach (var msg in messages)
    {
        if (msg.Role == MessageRole.Agent)
        {
            foreach (var contentItem in msg.ContentItems)
            {
                if (contentItem is MessageTextContent textContent)
                    return textContent.Text;
            }
        }
    }
    return string.Empty;
}

// ── エージェントの作成 ──────────────────────────────────────

// 1. トピック分類エージェント
var routingAgent = await agentsClient.CreateAgentAsync(
    model: "gpt-4.1",
    name: "routing-agent",
    instructions: "入力されたトピックを以下の選択肢に分類してください。選択肢の回答のみを返答してください。\n- Microsoft\n- Other"
);
Console.WriteLine($"routing-agent を作成しました (ID: {routingAgent.Value.Id})");

// 2. Microsoft 情報調査エージェント
var microsoftAgent = await agentsClient.CreateAgentAsync(
    model: "gpt-4.1",
    name: "microsoft-agent",
    instructions: "マイクロソフトの製品やサービスについての情報を収集して、分かりやすく回答してください。"
);
Console.WriteLine($"microsoft-agent を作成しました (ID: {microsoftAgent.Value.Id})");

// 3. 一般情報エージェント
var generalAgent = await agentsClient.CreateAgentAsync(
    model: "gpt-4.1",
    name: "general-agent",
    instructions: "ユーザーの質問に対して、幅広いトピックについて分かりやすく回答してください。"
);
Console.WriteLine($"general-agent を作成しました (ID: {generalAgent.Value.Id})");

// ── ワークフローの実行 ──────────────────────────────────────

var userInput = "Azure AI Foundry の最新機能について教えてください。";
Console.WriteLine($"\nユーザーの入力: {userInput}");

// ステップ 1: トピック分類
Console.WriteLine("\n--- ステップ 1: トピック分類 ---");
var routingResult = await RunAgentAsync(agentsClient, routingAgent.Value.Id, userInput);
Console.WriteLine($"分類結果: {routingResult}");

// ステップ 2: 分類結果に応じてエージェントを選択
Console.WriteLine("\n--- ステップ 2: エージェントの選択と実行 ---");
string finalResponse;
if (routingResult.Contains("Microsoft"))
{
    Console.WriteLine("Microsoft 情報調査エージェントを使用します。");
    finalResponse = await RunAgentAsync(agentsClient, microsoftAgent.Value.Id, userInput);
}
else
{
    Console.WriteLine("一般情報エージェントを使用します。");
    finalResponse = await RunAgentAsync(agentsClient, generalAgent.Value.Id, userInput);
}

Console.WriteLine($"\n最終応答:\n{finalResponse}");

// ── クリーンアップ ──────────────────────────────────────────
await agentsClient.DeleteAgentAsync(routingAgent.Value.Id);
await agentsClient.DeleteAgentAsync(microsoftAgent.Value.Id);
await agentsClient.DeleteAgentAsync(generalAgent.Value.Id);
Console.WriteLine("\n全エージェントを削除しました。");
```

### 4-2. ワークフローの実行

```bash
dotnet run
```

ワークフローが順に実行され、トピック分類の結果に応じて適切なエージェントが選択されて回答が生成されます。

![](./images/3-4-01.png)

> **動作の流れ**
> 1. ユーザーの入力を routing-agent が「Microsoft」または「Other」に分類します。
> 2. 分類結果に応じて、microsoft-agent または general-agent にユーザーの入力を渡します。
> 3. 選択されたエージェントが最終的な回答を生成します。

### 4-3. 複数ターンの会話への対応

同じスレッドを使い回すことで、エージェントとの複数ターンの会話を実現できます。

`Program.cs` の内容を以下に書き換えます。

```csharp
using Azure.AI.Projects;
using Azure.Identity;

var connectionString = "<接続文字列>";
var projectClient = new AIProjectClient(connectionString, new DefaultAzureCredential());
var agentsClient = projectClient.GetAgentsClient();

// エージェントの作成
var agent = await agentsClient.CreateAgentAsync(
    model: "gpt-4.1",
    name: "multi-turn-agent",
    instructions: "あなたは会話の文脈を理解する AI アシスタントです。前の会話を踏まえて回答してください。"
);
Console.WriteLine($"エージェントを作成しました: {agent.Value.Name}");

// スレッドを 1 つ作成して会話全体で使い回す
var thread = await agentsClient.CreateThreadAsync();
Console.WriteLine($"スレッドを作成しました: {thread.Value.Id}\n");

var questions = new[]
{
    "Azure AI Foundry とは何ですか？",
    "それはどのようなユースケースに向いていますか？",
    "始めるにはどうすればいいですか？",
};

for (int i = 0; i < questions.Length; i++)
{
    var question = questions[i];
    Console.WriteLine($"[ターン {i + 1}] ユーザー: {question}");

    await agentsClient.CreateMessageAsync(thread.Value.Id, MessageRole.User, question);

    var run = await agentsClient.CreateRunAsync(thread.Value.Id, agent.Value.Id);
    do
    {
        await Task.Delay(TimeSpan.FromSeconds(1));
        run = await agentsClient.GetRunAsync(thread.Value.Id, run.Value.Id);
    } while (run.Value.Status == RunStatus.Queued || run.Value.Status == RunStatus.InProgress);

    var messages = agentsClient.GetMessagesAsync(thread.Value.Id);
    await foreach (var msg in messages)
    {
        if (msg.Role == MessageRole.Agent)
        {
            foreach (var contentItem in msg.ContentItems)
            {
                if (contentItem is MessageTextContent textContent)
                {
                    Console.WriteLine($"[ターン {i + 1}] エージェント: {textContent.Text}\n");
                }
            }
            break;
        }
    }
}

// クリーンアップ
await agentsClient.DeleteAgentAsync(agent.Value.Id);
Console.WriteLine("エージェントを削除しました。");
```

```bash
dotnet run
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

`Program.cs` の内容を以下に書き換えます。`<エージェント ID>` の部分を 5-1 でコピーしたエージェント ID に置き換えます。

```csharp
using Azure.AI.Projects;
using Azure.Identity;

var connectionString = "<接続文字列>";
var projectClient = new AIProjectClient(connectionString, new DefaultAzureCredential());
var agentsClient = projectClient.GetAgentsClient();

var existingAgentId = "<エージェント ID>";

// 既存エージェントの取得
var agent = await agentsClient.GetAgentAsync(existingAgentId);
Console.WriteLine($"エージェントを取得しました: {agent.Value.Name} (ID: {agent.Value.Id})");

// スレッドの作成とメッセージの送信
var thread = await agentsClient.CreateThreadAsync();
await agentsClient.CreateMessageAsync(
    threadId: thread.Value.Id,
    role: MessageRole.User,
    content: "こんにちは！あなたは何ができますか？"
);

// エージェントの実行
var run = await agentsClient.CreateRunAsync(thread.Value.Id, agent.Value.Id);
do
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    run = await agentsClient.GetRunAsync(thread.Value.Id, run.Value.Id);
} while (run.Value.Status == RunStatus.Queued || run.Value.Status == RunStatus.InProgress);

Console.WriteLine($"実行ステータス: {run.Value.Status}");

// 応答メッセージの取得
var messages = agentsClient.GetMessagesAsync(thread.Value.Id);
await foreach (var msg in messages)
{
    if (msg.Role == MessageRole.Agent)
    {
        foreach (var contentItem in msg.ContentItems)
        {
            if (contentItem is MessageTextContent textContent)
            {
                Console.WriteLine($"\nエージェントの応答:\n{textContent.Text}");
            }
        }
        break;
    }
}
```

```bash
dotnet run
```

![](./images/3-5-02.png)

> 既存エージェントを呼び出す場合、エージェント自体は削除しないよう注意してください。ポータルで作成・管理しているエージェントは、SDK からも参照・操作できます。

---

## まとめ

本ハンズオンでは、Azure AI Projects SDK (C#) を使って以下を学びました。

| 内容 | 概要 |
|---|---|
| シンプルなエージェントの作成 | `CreateAgentAsync` / `CreateThreadAsync` / `CreateRunAsync` の基本的な使い方 |
| ツールの追加 | コードインタープリター (`CodeInterpreterToolDefinition`) とファイル検索 (`FileSearchToolDefinition`) をエージェントに追加する方法 |
| マルチエージェントワークフロー | 複数のエージェントを組み合わせてトピックに応じた処理を行う方法 |
| 複数ターンの会話 | スレッドを使い回して文脈を維持した会話を行う方法 |
| 既存エージェントの利用 | ポータルで作成したエージェントを `GetAgentAsync` で取得・呼び出す方法 |

SDK を使うことで、エージェントの作成・実行・管理をプログラムから柔軟に行えます。ポータル上での操作と組み合わせることで、より高度な AI アプリケーションを開発できます。

### 参考リンク

- [Azure AI Projects SDK (.NET) ドキュメント](https://learn.microsoft.com/azure/ai-foundry/how-to/develop/sdk-overview)
- [Azure AI Agent Service クイックスタート (.NET)](https://learn.microsoft.com/azure/ai-services/agents/quickstart?pivots=programming-language-csharp)
- [Azure.AI.Projects NuGet パッケージ](https://www.nuget.org/packages/Azure.AI.Projects)
