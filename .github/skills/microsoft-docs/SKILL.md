---
name: microsoft-docs
description: "Microsoft Learn と Microsoft 技術ドキュメントを日本語環境で調査・要約・参照するための Skill。Azure、.NET、C#、ASP.NET Core、Agent Framework、Aspire、Power Platform、Microsoft 365、Windows、GitHub、VS Code、コードサンプル検索、ja-jp/en-us 差分確認を扱う。"
license: MIT
---

# Microsoft Docs — 日本語環境向け Skill

この Skill は、Microsoft Learn と Microsoft 技術ドキュメントを調査し、正確な根拠に基づいて日本語で回答するために使う。Azure、.NET、C#、ASP.NET Core、Microsoft Agent Framework、Microsoft.Extensions.AI、Microsoft 365、Power Platform、Windows、Visual Studio、GitHub 連携などを対象にする。

> **基本方針:** 公式 Microsoft Learn を一次情報として使う。日本語環境では `ja-jp` ページを優先するが、翻訳遅延・訳語の不自然さ・新機能の反映遅れが疑われる場合は `en-us` ページも確認する。コードを提示する場合は、公式コードサンプルや対象 SDK バージョンに基づく。

---

## 1. まず確認すること

| 確認項目 | 観点 |
|---|---|
| 技術領域 | Azure、.NET、C#、Agent Framework、M365、Power Platform など |
| 目的 | 概要、quickstart、tutorial、API reference、制限、移行、トラブルシュート |
| バージョン | .NET 8/9/10、C# 13/14、Azure SDK version、CLI version など |
| 言語 | C#、PowerShell、Azure CLI、TypeScript、Python など |
| 日本語要件 | `ja-jp` ページを優先するか、英語原文確認が必要か |
| 実行環境 | Windows PowerShell、Linux、コンテナ、Azure、GitHub Actions |
| 公式性 | Learn、API reference、GitHub repo、release notes のどれが根拠か |
| 鮮度 | 最新 SDK / preview / breaking changes / deprecated API |

ユーザーが URL を示した場合は、そのページを直接 fetch してから回答する。

---

## 2. Microsoft Learn MCP の使い分け

Microsoft Learn 上の情報は、次の tools を優先する。

| Tool | 用途 |
|---|---|
| `microsoft_docs_search` | Microsoft Learn 全体から概念、手順、制限、トラブルシュートを検索 |
| `microsoft_code_sample_search` | 公式ドキュメント内のコード例を検索。言語指定を推奨 |
| `microsoft_docs_fetch` | 特定 URL の全文を取得。tutorial、手順、前提条件、制限確認に使う |

使い方:

1. まず `microsoft_docs_search` で候補を見つける。
2. 重要なページは `microsoft_docs_fetch` で全文を確認する。
3. コードを書く場合は `microsoft_code_sample_search` で公式サンプルを確認する。
4. 回答には、必要な前提条件、バージョン、制限、公式 URL を含める。

---

## 3. 日本語ページと英語ページ

日本語環境では `https://learn.microsoft.com/ja-jp/...` を優先する。

ただし、次の場合は `en-us` も確認する。

- preview / latest / breaking changes に関する情報。
- 日本語ページの翻訳が不自然。
- UI 名、CLI option、API 名が翻訳されていて曖昧。
- `ja-jp` ページが古い、または内容が欠けている。
- エラーメッセージや SDK の型名を正確に確認したい。

回答では、翻訳差分がありそうな場合にその旨を明記する。

```text
日本語ページでは「フィールド」と訳されていますが、API 名は `field` のままです。
コード上では英語の識別子を使用してください。
```

---

## 4. CLI 代替手段

Microsoft Learn MCP が使えない場合は、`mslearn` CLI を使う。

PowerShell:

```powershell
npx @microsoft/learn-cli search "Azure Functions .NET isolated worker configuration"
npx @microsoft/learn-cli code-search "BlobClient UploadAsync" --language csharp
npx @microsoft/learn-cli fetch "https://learn.microsoft.com/ja-jp/dotnet/csharp/whats-new/csharp-14"
```

global install:

```powershell
npm install -g @microsoft/learn-cli
mslearn search "Microsoft Agent Framework workflow orchestration"
mslearn code-search "IChatClient ChatClientAgent" --language csharp
mslearn fetch "https://learn.microsoft.com/ja-jp/dotnet/"
```

JSON 出力:

```powershell
mslearn search "TimeProvider FakeTimeProvider" --json | ConvertFrom-Json
```

---

## 5. 効果的な検索クエリ

曖昧な語だけで検索しない。技術名、バージョン、目的、言語を含める。

悪い例:

```text
Azure Functions
Agent Framework
Blob
```

良い例:

```text
Azure Functions .NET 8 isolated worker dependency injection
C# 14 null conditional assignment field keyword
Microsoft Agent Framework ChatClientAgent IChatClient C#
TimeProvider FakeTimeProvider xUnit testing
Azure Storage BlobClient UploadAsync C#
ASP.NET Core localization IStringLocalizer ja-JP
```

コードサンプル検索では language を指定する。

| 言語 | 指定値 |
|---|---|
| C# | `csharp` |
| PowerShell | `powershell` |
| Azure CLI | `azurecli` |
| TypeScript | `typescript` |
| JavaScript | `javascript` |
| Python | `python` |

---

## 6. コードを提示するときのルール

Microsoft / Azure 関連コードを提示する場合:

- 公式コードサンプルを検索する。
- SDK package 名と version 前提を確認する。
- preview API か stable API かを明記する。
- secret をコードに直書きしない。
- Azure 認証は可能なら `DefaultAzureCredential` を優先する。
- Windows PowerShell の手順では `\` path と PowerShell syntax を使う。
- 日本語文字列は UTF-8 を前提にする。
- .NET の時刻依存コードでは `TimeProvider` を優先する。
- テスト例は xUnit を基本にする。

例:

```text
このコードは .NET 8 以降と Azure.Identity を前提にしています。
ローカルでは Azure CLI または Visual Studio の認証、Azure 上では managed identity を使います。
```

---

## 7. 対象外・別ソースを優先するケース

Microsoft 関連でも、learn.microsoft.com 以外が一次情報になるものがある。

| 領域 | 優先ソース |
|---|---|
| .NET Aspire | `aspire.dev`、Aspire MCP、`dotnet/aspire` |
| GitHub | `docs.github.com`、`cli.github.com`、GitHub MCP / gh CLI |
| VS Code | `code.visualstudio.com`、VS Code API docs |
| Fluent UI Blazor | `microsoft/fluentui-blazor`、公式 demo |
| OSS repository の最新仕様 | 公式 GitHub repo、release notes |
| NuGet package の version | NuGet.org、公式 repo、release notes |

ただし、Microsoft Learn に該当する概要やチュートリアルがある場合は併用する。

---

## 8. .NET / C# ドキュメント調査

.NET / C# では次を確認する。

- 対象 .NET version
- C# language version
- API reference と conceptual docs の両方
- breaking changes
- obsolete / deprecated
- cross-platform differences
- nullable annotations
- trimming / AOT 対応

よく使うクエリ:

```text
.NET TimeProvider FakeTimeProvider testing
C# 13 new features params collections lock object
C# 14 new features extension members field keyword
ASP.NET Core options pattern ValidateOnStart
Microsoft.Extensions.Logging structured logging
```

---

## 9. Azure ドキュメント調査

Azure では次を確認する。

- region availability。日本向けでは Japan East / Japan West を候補にする。
- pricing tier / SKU / quota / limit
- RBAC role
- managed identity 対応
- private endpoint / networking
- data residency
- SDK package と API version
- Azure CLI / PowerShell command
- Terraform / Bicep が必要か

回答には、必要に応じて次を分けて書く。

- Portal 手順
- Azure CLI
- Azure PowerShell
- Bicep / IaC
- .NET SDK

---

## 10. Microsoft Agent Framework 調査

Agent Framework は Learn の tutorial / concept と、GitHub repo の API 詳細を併用する。

確認すること:

- `Microsoft.Agents.AI`
- `Microsoft.Extensions.AI`
- `IChatClient`
- `ChatClientAgent`
- orchestration / workflow
- tool calling
- structured output
- observability / OpenTelemetry
- provider: Azure OpenAI、Foundry、OpenAI など
- package が preview / rc / stable か

検索例:

```text
Microsoft Agent Framework ChatClientAgent IChatClient C#
Agent Framework workflow orchestration handoff group chat
Agent Framework OpenTelemetry tracing .NET
```

新規 AI 操作の実装方針では、Semantic Kernel ではなく Microsoft Agent Framework を優先する。

---

## 11. 回答品質のルール

回答には必要に応じて次を含める。

- 推奨結論
- 対象 version
- 前提条件
- 最小コードまたはコマンド
- 注意点 / 制限
- 公式 URL
- 日本語ページと英語ページの差分がある場合の補足

不確かな場合は断言しない。

```text
この機能は preview のため、API 名や package version が変わる可能性があります。
```

---

## 12. よくある調査パターン

### 公式サンプル付きで C# コードを出す

1. `microsoft_code_sample_search` で対象 API を検索する。
2. `microsoft_docs_fetch` で該当ページの前提条件を確認する。
3. package、using、認証、例外、制限を明記する。
4. 日本語で補足する。

### Learn URL を要約する

1. URL を `microsoft_docs_fetch` で取得する。
2. 対象読者向けに要点を整理する。
3. 手順、前提条件、注意点、コードを分ける。
4. 古い情報や preview 表記がないか確認する。

### エラーを調査する

1. エラーメッセージを正確に検索する。
2. SDK / CLI / runtime version を確認する。
3. 公式 troubleshooting / known issues を探す。
4. 再現条件と修正手順を示す。

---

## 13. セキュリティとコンプライアンス

- secret、token、connection string を回答例に含めない。
- credential は環境変数、User Secrets、Key Vault、managed identity を使う。
- Azure RBAC は最小権限を推奨する。
- サンプルの tenant ID、subscription ID、endpoint は placeholder にする。
- public issue / docs / prompt に顧客情報を含めない。
- ライセンスや preview 条項が関係する場合は確認を促す。

---

## 14. レビュー時チェックリスト

- [ ] 公式 Microsoft Learn を根拠にしている
- [ ] 必要に応じて `ja-jp` と `en-us` を確認している
- [ ] バージョンと前提条件が明記されている
- [ ] コードは公式サンプルや対象 SDK に沿っている
- [ ] preview / deprecated / breaking changes を確認している
- [ ] secret を含めていない
- [ ] Windows PowerShell と日本語環境の注意を反映している
- [ ] 公式 URL を示している
- [ ] Learn 以外が一次情報の領域では適切なソースを使っている

---

## 15. 公式参照

| リソース | URL |
|---|---|
| Microsoft Learn | https://learn.microsoft.com/ja-jp/ |
| .NET documentation | https://learn.microsoft.com/ja-jp/dotnet/ |
| C# documentation | https://learn.microsoft.com/ja-jp/dotnet/csharp/ |
| Azure documentation | https://learn.microsoft.com/ja-jp/azure/ |
| ASP.NET Core documentation | https://learn.microsoft.com/ja-jp/aspnet/core/ |
| Microsoft Agent Framework | https://learn.microsoft.com/agent-framework/ |
| Microsoft 365 documentation | https://learn.microsoft.com/ja-jp/microsoft-365/ |
| Power Platform documentation | https://learn.microsoft.com/ja-jp/power-platform/ |
| Microsoft Learn CLI | https://www.npmjs.com/package/@microsoft/learn-cli |
