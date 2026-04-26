---
name: microsoft-agent-framework
description: "Microsoft Agent Framework を日本語環境で設計・実装・レビュー・移行するための Skill。.NET/C#、Python、IChatClient、ChatClientAgent、agents、workflows、tool calling、structured output、OpenTelemetry、Microsoft.Extensions.AI、Azure OpenAI、Microsoft Foundry、Semantic Kernel / AutoGen からの移行を扱う。"
license: MIT
---

# Microsoft Agent Framework — 日本語環境向け Skill

この Skill は、Microsoft Agent Framework を使った agent、workflow、tool calling、multi-agent orchestration、移行、レビュー、トラブルシュートを支援するために使う。日本語環境では説明・設計メモ・運用手順は日本語を基本にし、コード識別子・API 名・設定キーは英語を維持する。

> **基本方針:** 新規の AI 操作は Microsoft Agent Framework を既定にする。Semantic Kernel や AutoGen は移行元として扱い、最新の公式ドキュメントとサンプルを確認してから実装する。public preview や release candidate の API は変わる可能性があるため、package version と公式 docs を必ず確認する。

---

## 1. まず判断すること

| 確認項目 | 観点 |
|---|---|
| 対象言語 | .NET/C#、Python、または混在 |
| 実装対象 | agent、workflow、tool、middleware、context provider、migration |
| provider | Microsoft Foundry、Azure OpenAI、OpenAI、その他 `IChatClient` |
| 認証 | `DefaultAzureCredential`、API key、managed identity |
| 状態管理 | chat history、thread、session、checkpoint |
| オーケストレーション | sequential、concurrent、handoff、group chat、magentic、custom workflow |
| tool 実行 | function calling、approval、権限、監査 |
| 出力形式 | text、structured output、domain model |
| 観測性 | logging、OpenTelemetry、tracing、metrics |
| テスト | xUnit / fake `IChatClient` / deterministic test |
| 日本語要件 | prompt、UI、ログ、文字コード、個人情報 |

対象言語が曖昧な場合は workspace を確認する。`.csproj`、`.sln`、`.cs` があれば .NET workflow、`.py`、`pyproject.toml`、`requirements.txt` があれば Python workflow を優先する。混在時は変更対象ファイルに合わせる。

---

## 2. References

| Reference | 用途 |
|---|---|
| `references/dotnet.md` | .NET / C# の package、DI、`IChatClient`、`ChatClientAgent`、xUnit |
| `references/python.md` | Python の package、async、型ヒント、環境管理 |
| `references/workflows-and-safety.md` | workflow、tool approval、security、observability、運用設計 |

---

## 3. 必ず最新ドキュメントを確認する

Microsoft Agent Framework は変化が速い。作業前に公式情報を確認する。

優先ソース:

1. Microsoft Learn: https://learn.microsoft.com/agent-framework/overview/agent-framework-overview
2. Learn の agent / workflow / migration / observability ページ
3. GitHub repository: https://github.com/microsoft/agent-framework
4. .NET samples: https://github.com/microsoft/agent-framework/tree/main/dotnet/samples
5. Python samples: https://github.com/microsoft/agent-framework/tree/main/python/samples

Microsoft Learn MCP が使える場合:

- `microsoft_docs_search` で最新 guidance を検索する。
- `microsoft_docs_fetch` で重要ページを全文確認する。
- C# コードを提示する場合は `microsoft_code_sample_search` も使う。

検索例:

```text
Microsoft Agent Framework ChatClientAgent IChatClient C#
Microsoft Agent Framework workflows sequential orchestration C#
Agent Framework tool calling structured output OpenTelemetry
Agent Framework migration from Semantic Kernel C#
```

---

## 4. Agent と Workflow の使い分け

| 用途 | 推奨 |
|---|---|
| 会話、ツール利用、動的判断、アドホックな計画 | Agent |
| 手順が明確な業務プロセス | Workflow |
| multi-step の定型処理 | Workflow |
| LLM が次の行動を選ぶ必要がある | Agent |
| human-in-the-loop、承認、checkpoint、復元 | Workflow |
| 複数 agent の順次・並列・handoff | Workflow orchestration |
| 既存 API から単一 agent として扱いたい複雑処理 | Workflow as Agent |

原則:

- 自律判断が必要なら agent。
- 制御フローを明示したいなら workflow。
- 業務上の監査・承認・再開が重要なら workflow。
- 単純な 1 回の chat completion を無理に agent 化しない。

---

## 5. .NET / C# 既定方針

.NET プロジェクトでは `references/dotnet.md` に従う。

基本 package:

```powershell
dotnet add package Microsoft.Agents.AI
```

最小例:

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

ChatClientAgent agent = new(
    chatClient,
    instructions: "あなたは社内業務を支援するアシスタントです。");
```

方針:

- `Microsoft.Extensions.AI.IChatClient` を provider 抽象として使う。
- `ChatClientAgent` は `IChatClient` から構成する。
- DI、Options、logging、`DefaultAzureCredential` など .NET 標準パターンに合わせる。
- async / await と `CancellationToken` を徹底する。
- 時刻依存の tool や business logic は `TimeProvider` を使う。
- テストは xUnit を使い、実 model 呼び出しに依存しない。

---

## 6. Python 既定方針

Python プロジェクトでは `references/python.md` に従う。

基本 package:

```powershell
pip install agent-framework
```

方針:

- async を基本にする。
- 型ヒントを付ける。
- `pyproject.toml` / virtual environment / requirements 管理に従う。
- prompt、tool、workflow 定義を分離する。
- secret は環境変数や secret store から取得する。
- 最新 samples を確認してから API を使う。

---

## 7. Tool calling と権限

Tool は AI が呼び出せる「権限付き操作」なので、安全に設計する。

確認すること:

- tool の責務が小さい。
- 引数が明示的で型付けされている。
- 入力検証がある。
- 権限チェックがある。
- destructive operation には human approval を挟む。
- 実行結果が監査ログに残る。
- secret や個人情報を model に渡さない。

避ける:

- 任意 SQL 実行 tool。
- 任意 shell 実行 tool。
- 無制限ファイル読み書き tool。
- 認可なしの GitHub / Azure / DB 更新 tool。
- AI 応答を検証せずに外部送信や DB 更新へ使う。

---

## 8. Prompt / instructions 設計

日本語環境では、instructions は日本語でよい。ただし、API 名、型名、field 名、JSON schema は英語を維持する。

含めるべき内容:

- agent の役割。
- できること / できないこと。
- 利用可能 tools と使う条件。
- 出力形式。
- 日本語で回答する条件。
- 機密情報を扱わない方針。
- 不明点を確認する条件。

例:

```text
あなたは社内サポート担当の AI エージェントです。
回答は日本語で行ってください。
根拠が不足している場合は推測せず、必要な追加情報を確認してください。
個人情報、API キー、接続文字列を出力しないでください。
```

prompt はコードに散らばらせず、version 管理できる場所に集約する。

---

## 9. Structured output

業務処理へ渡す出力は、可能な限り structured output にする。

推奨:

- record / class / Pydantic model などで契約を定義する。
- 必須 field と optional field を明確にする。
- AI 出力を domain validation に通す。
- 不正な出力は再試行または明示エラーにする。

避ける:

- 自然文を正規表現で無理に parse する。
- AI 応答をそのまま SQL / API request に使う。
- null / empty の扱いを曖昧にする。

---

## 10. 状態管理

会話や workflow の状態を扱う場合:

- chat history の保存範囲を決める。
- 個人情報や機密情報を保存しない、またはマスキングする。
- session / thread / checkpoint の寿命を決める。
- long-running workflow は再開可能性を設計する。
- tenant / user / conversation の境界を混同しない。

Workflow as Agent のように session serialization / resumption を使う場合は、永続化先、暗号化、schema 変更、削除ポリシーを決める。

---

## 11. Observability

運用では、AI 呼び出しがブラックボックスにならないようにする。

確認すること:

- `ILogger<T>` / structured logging。
- OpenTelemetry tracing。
- model provider、deployment、latency、token usage、tool call count。
- tool approval / rejection。
- workflow edge / status / checkpoint。
- error type と retry 可否。

ログに含めないもの:

- API key。
- access token。
- connection string。
- 個人情報。
- 顧客データ。
- prompt 全文や応答全文が機密を含む場合。

---

## 12. Migration from Semantic Kernel / AutoGen

移行では、いきなり全面書き換えしない。

手順:

1. 既存の agent / plugin / tool / workflow の責務を整理する。
2. 外部 observable behavior を固定する。
3. Semantic Kernel / AutoGen 固有型を application 層から切り離す。
4. `IChatClient`、`ChatClientAgent`、workflow へ段階的に移行する。
5. テストを先に用意し、挙動を維持する。
6. 新しい Agent Framework pattern は移行後に段階導入する。

公式 migration guide:

- https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/
- https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/

---

## 13. セキュリティと responsible AI

必ず検討すること:

- prompt injection。
- tool abuse。
- data exfiltration。
- 最小権限。
- human approval。
- audit log。
- content filtering。
- PII / secret masking。
- tenant isolation。
- rate limit / quota。
- timeout / retry / circuit breaker。

ユーザー入力、検索結果、外部ドキュメント、tool 出力は信頼境界が異なる。system / developer instructions と同じ信頼度で扱わない。

---

## 14. テスト方針

.NET では xUnit を使う。

テスト対象:

- instructions の組み立て。
- tool 入力検証。
- tool 権限チェック。
- structured output validation。
- workflow routing。
- error handling。
- timeout / cancellation。
- `TimeProvider` による時刻依存。

避ける:

- 単体テストで実 model に依存する。
- 現在時刻や外部 API に依存する。
- secret が必要なテストを既定にする。

必要に応じて integration test と manual evaluation を分ける。

---

## 15. レビュー時チェックリスト

- [ ] 対象言語に合った reference を使っている
- [ ] 最新の公式 docs / samples を確認している
- [ ] 新規 AI 操作は Microsoft Agent Framework を使っている
- [ ] Semantic Kernel / AutoGen pattern を新規既定にしていない
- [ ] `IChatClient` / provider 抽象が application 層に適切に隠蔽されている
- [ ] agent と workflow の使い分けが妥当
- [ ] tool が最小権限で、入力検証と監査がある
- [ ] structured output と validation がある
- [ ] secret / 個人情報を prompt、ログ、履歴に残していない
- [ ] OpenTelemetry / structured logging を検討している
- [ ] .NET テストは xUnit で実 model に依存しない
- [ ] 時刻依存処理は `TimeProvider` を使っている

---

## 16. 公式参照

| リソース | URL |
|---|---|
| Microsoft Agent Framework overview | https://learn.microsoft.com/agent-framework/overview/agent-framework-overview |
| Agent types | https://learn.microsoft.com/agent-framework/agents/ |
| Agent pipeline | https://learn.microsoft.com/agent-framework/agents/agent-pipeline |
| Workflows | https://learn.microsoft.com/agent-framework/workflows/ |
| Semantic Kernel migration | https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/ |
| AutoGen migration | https://learn.microsoft.com/agent-framework/migration-guide/from-autogen/ |
| Microsoft.Extensions.AI | https://learn.microsoft.com/dotnet/ai/microsoft-extensions-ai |
| GitHub repository | https://github.com/microsoft/agent-framework |
