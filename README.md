# DotnetTemplate

日本語環境の .NET / C# 開発で GitHub Copilot を安全かつ一貫して使うための、リポジトリ向け Copilot customization テンプレートです。

このリポジトリには、Copilot Agent Skills、Custom Agents、Pull Request テンプレートを含めています。.NET / C# の実装、レビュー、テスト、Microsoft Learn 調査、GitHub CLI 操作、アクセシビリティ確認を、日本語チームの運用に合わせて支援することを目的にしています。

## 主な内容

| 種別 | パス | 用途 |
|---|---|---|
| Agent Skills | `.github\skills\<skill-name>\SKILL.md` | Copilot がタスクに応じて読み込む専門手順・規約・参照情報 |
| Custom Agents | `.github\agents\*.agent.md` | 役割ごとの専門 Agent 定義 |
| Pull Request template | `.github\pull_request_template.md` | .NET 向け PR 記述・確認チェックリスト |

## 想定する開発方針

- .NET / C# を前提にします。
- テストは xUnit を基本にします。
- 現在時刻の取得は `DateTime.Now` / `DateTimeOffset.Now` / `DateTime.UtcNow` を直接使わず、`TimeProvider` を使います。
- AI 操作は Microsoft Agent Framework を優先します。
- 日本語環境として UTF-8、Windows / PowerShell、`ja-JP`、`Asia/Tokyo` を考慮します。
- ASP.NET Core / Razor / Blazor / MVC / Fluent UI Blazor ではアクセシビリティをレビュー観点に含めます。

## 既定アーキテクチャ

ユーザーから明示的な指定がない場合、新規 .NET アプリケーションでは次を既定として採用します。既存プロジェクトでは既存構成を優先し、必要な差分だけ適用します。

| 項目 | 既定 |
|---|---|
| Runtime / Framework | 最新の安定版 .NET SDK と ASP.NET Core。preview / RC は明示指定がある場合だけ使用 |
| UI | Blazor Web App の Interactive Server render mode |
| Component library | Fluent UI Blazor (`Microsoft.FluentUI.AspNetCore.Components`) |
| App orchestration | .NET Aspire の AppHost と ServiceDefaults |
| Data access | 永続化が必要な場合のみ EF Core |
| Database | DB が必要な場合は SQL Server を第一候補 |
| Testing | xUnit。時刻依存テストは `FakeTimeProvider` |
| Time / locale | 内部処理は UTC、表示・入力は `ja-JP` / `Asia/Tokyo` を考慮 |
| Configuration | Options pattern、User Secrets、環境変数。secret はリポジトリに含めない |
| Observability | Aspire ServiceDefaults と OpenTelemetry を優先 |
| Accessibility | Fluent UI Blazor / Blazor UI では WCAG 2.2、JIS X 8341-3、keyboard / focus / label / ARIA を確認 |

既定の solution 構成は、規模に応じて最小構成から始めます。

| 用途 | 例 |
|---|---|
| Web UI | `src\<ProductName>.Web` |
| Aspire AppHost | `src\<ProductName>.AppHost` |
| Aspire ServiceDefaults | `src\<ProductName>.ServiceDefaults` |
| Tests | `tests\<ProductName>.Tests` |
| Domain / Application / Infrastructure | 複雑な業務ロジックや外部依存が増えた場合だけ分割 |

## Custom Agents

`.github\agents` には、.NET / C# 開発向けの役割分担を定義しています。

| Agent | 役割 | 権限の考え方 |
|---|---|---|
| `.NET 開発オーケストレーター` | .NET / C# の実装、修正、テスト追加、リファクタリング、レビュー反映依頼で最初に呼ばれる入口 Agent | 自身は書き換えず、必要な Agent を統括 |
| `.NET コードライター` | C# / .NET コード、xUnit テスト、関連ファイルを実装・修正する Writer | 書き込み担当 |
| `.NET コードレビュアー` | 差分を読み取り、bug、security、設計、テスト不足などをレビュー | 読み取り専用 |
| `.NET レビューチェッカー` | Reviewer とアクセシビリティ専門家の指摘が正当かを分類 | 読み取り専用 |
| `アクセシビリティ専門家` | ASP.NET Core UI 差分を WCAG / JIS / 日本語 UX 観点で確認 | 読み取り専用 |
| `C#/.NET 専門家` | Orchestrator から必要時に呼ばれる .NET / C# 固有判断の相談役 | 読み取り専用 |

### Agent の基本フロー

1. `.NET 開発オーケストレーター` がユーザー要求を整理します。
2. 必要に応じて `C#/.NET 専門家` へ設計、API、互換性、性能、セキュリティの助言を求めます。
3. `.NET コードライター` が実装とテストを行います。
4. `.NET コードレビュアー` が差分をレビューします。
5. ASP.NET Core UI 変更がある場合は `アクセシビリティ専門家` が専門レビューします。
6. `.NET レビューチェッカー` が指摘の正当性を確認します。
7. 正当な未解決指摘がなくなるまで Writer が修正し、Orchestrator が収束判定します。

## Agent Skills

`.github\skills` には、Copilot が関連タスクで参照できる Skill を配置しています。

| Skill | 用途 |
|---|---|
| `aspire` | .NET Aspire の AppHost、分散アプリ構成、サービス検出、統合、ダッシュボード、テスト |
| `csharp-docs` | C# / .NET の XML ドキュメントコメント、API ドキュメント、サンプルコード |
| `csharp-xunit` | xUnit による .NET / C# テスト設計、実装、保守 |
| `dotnet-best-practices` | .NET / C# の設計、DI、設定、ログ、例外、非同期、TimeProvider、Microsoft Agent Framework |
| `dotnet-design-pattern-review` | 設計パターン、SOLID、DI、Repository、Provider、Factory、Command などのレビュー |
| `dotnet-timezone` | `TimeProvider`、`TimeZoneInfo`、`DateTimeOffset`、日本時間、タイムゾーン処理 |
| `ef-core` | Entity Framework Core の DbContext、Entity、LINQ、Migration、性能、テスト |
| `fluentui-blazor` | Fluent UI Blazor による UI 実装、DataGrid、レイアウト、テーマ、アクセシビリティ |
| `gh-cli` | GitHub CLI による repository、issue、PR、Actions、release、API 操作 |
| `git-commit` | Git commit 作成、Conventional Commits、差分確認、機密情報混入防止 |
| `github-issues` | GitHub Issues の作成、検索、triage、ラベル、assignee、milestone、Projects 連携 |
| `microsoft-agent-framework` | Microsoft Agent Framework の設計、実装、レビュー、移行 |
| `microsoft-code-reference` | Microsoft / Azure / .NET SDK の API、公式コードサンプル、メソッド署名確認 |
| `microsoft-docs` | Microsoft Learn と Microsoft 技術ドキュメントの調査、要約、参照 |

## 使い方

このリポジトリをベースに、対象プロジェクトへ `.github` 配下を配置します。

Copilot CLI や Copilot cloud agent で Custom Agents / Agent Skills が利用可能な環境では、`.github\agents` と `.github\skills` がリポジトリスコープの customization として扱われます。

.NET / C# の実装依頼では、`.NET 開発オーケストレーター` を入口にしてください。C# / .NET の専門判断が必要な場合は、Orchestrator が `C#/.NET 専門家` を相談役として利用します。

## 検証

Agent Skills の構成は GitHub CLI で dry run 検証できます。

```powershell
gh skill publish .github\skills --dry-run
```

Custom Agents の `tools` は GitHub Docs の custom agents configuration にある tool aliases を使います。

- `read`
- `search`
- `edit`
- `execute`
- `agent`
- `web`

読み取り専用 Agent には `edit` / `execute` を付与しない方針です。

## ライセンス

このリポジトリは MIT License です。詳細は `LICENSE` を参照してください。
