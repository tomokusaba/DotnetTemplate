---
name: aspire
description: ".NET Aspire の AppHost、分散アプリ構成、サービス検出、統合、ダッシュボード、テスト、デプロイ、トラブルシュートを日本語環境で支援するための Skill。Windows/PowerShell、日本語チーム運用、Azure Japan リージョン、UTF-8、ローカライズ要件を考慮する。"
license: MIT
---

# .NET Aspire — 日本語環境向け分散アプリ Skill

この Skill は、.NET Aspire を使った分散アプリケーションの作成、実行、構成、デバッグ、テスト、デプロイ、運用改善を支援する。回答と作業は原則として日本語で行い、日本語チームでの開発・レビュー・運用に適した判断を優先する。

> **基本方針:** AppHost は分散アプリ全体の「指揮者」。各サービスを直接実装する場所ではなく、サービスの起動順序、依存関係、接続情報、ヘルスチェック、監視をまとめて定義する場所として扱う。

---

## 1. まず確認すること

作業を始める前に、次を確認する。

| 確認項目 | 見る場所 / コマンド |
|---|---|
| .NET SDK バージョン | `global.json`, `dotnet --info` |
| Aspire CLI バージョン | `aspire --version` |
| AppHost プロジェクト | `*.AppHost`, `Program.cs`, `.csproj` |
| ServiceDefaults | `*.ServiceDefaults`, OpenTelemetry / health check 設定 |
| テストプロジェクト | `*.Tests`, `Aspire.Hosting.Testing` |
| コンテナ実行環境 | Docker Desktop, Podman, Rancher Desktop |
| シークレット管理 | User Secrets, 環境変数, CI/CD secret |
| デプロイ先 | Docker, Kubernetes, Azure Container Apps, Azure App Service |

既存プロジェクトでは、`global.json`、README、CI 設定、`Directory.Build.props` の方針を優先する。新規作成時は、利用可能な最新の安定版 .NET SDK と Aspire CLI を使う。

---

## 2. 日本語環境での回答・実装ルール

- 説明、コメント、PR 本文、手順は日本語を基本にする。
- コード内の識別子、API 名、ログの構造化フィールド名は英語を基本にする。
- 日本語文字列を扱うファイルは UTF-8 を前提にする。
- Windows 環境では PowerShell コマンドを優先し、パスは `\` 区切りで案内する。
- 日時を扱う場合は、保存形式は UTC、表示は `Asia/Tokyo` を基本に検討する。
- UI や通知文を追加する場合は、日本語リソース化や多言語対応の余地を確認する。
- Azure にデプロイする場合は、要件がなければ `Japan East` / `Japan West` を候補にする。ただし、サービス提供状況、価格、可用性要件を必ず確認する。
- シークレット、接続文字列、API キーはリポジトリに含めない。ローカルは User Secrets または環境変数、CI/CD は secret store を使う。

---

## 3. ドキュメント調査の優先順位

1. プロジェクト内の README、ADR、既存コード、CI 設定
2. Aspire CLI の MCP / docs search が使える場合はそれを優先
3. 公式ドキュメント: https://aspire.dev
4. 公式リポジトリ: https://github.com/dotnet/aspire
5. サンプル: https://github.com/dotnet/aspire-samples
6. Community Toolkit: https://github.com/CommunityToolkit/Aspire

CLI バージョン差分が疑われる場合は、必ず `aspire --version` と公式ドキュメントで確認する。

---

## 4. インストールと初期確認

### Windows PowerShell

```powershell
irm https://aspire.dev/install.ps1 | iex
aspire --version
dotnet --info
dotnet new install Aspire.ProjectTemplates
```

### macOS / Linux

```bash
curl -sSL https://aspire.dev/install.sh | bash
aspire --version
dotnet --info
dotnet new install Aspire.ProjectTemplates
```

テンプレートや CLI の更新は、プロジェクトの SDK / CI 方針と互換性を確認してから行う。

---

## 5. よく使う CLI

| コマンド | 用途 |
|---|---|
| `aspire new <template>` | Aspire テンプレートから新規作成 |
| `aspire init` | 既存アプリに Aspire 構成を追加 |
| `aspire run` | AppHost からローカル実行 |
| `aspire add <integration>` | Redis、PostgreSQL などの統合を追加 |
| `aspire publish` | デプロイ用成果物やマニフェストを生成 |
| `aspire deploy` | 対応ターゲットへデプロイ |
| `aspire mcp init` | AI アシスタント向け MCP 設定 |
| `aspire mcp start` | Aspire MCP サーバー起動 |

コマンドのオプションは CLI バージョンで変わる可能性があるため、詳細は `aspire <command> --help` を確認する。

---

## 6. AppHost 実装パターン

AppHost では、依存関係を明示し、接続情報は Aspire の service discovery と integration に任せる。

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres")
    .AddDatabase("appdb");

var redis = builder.AddRedis("cache");

var api = builder.AddProject<Projects.Api>("api")
    .WithReference(postgres)
    .WithReference(redis)
    .WaitFor(postgres)
    .WaitFor(redis);

builder.AddProject<Projects.Web>("web")
    .WithReference(api)
    .WaitFor(api);

builder.Build().Run();
```

### 判断基準

| 状況 | 推奨 |
|---|---|
| .NET プロジェクトを追加 | `AddProject<Projects.X>()` |
| Redis / PostgreSQL などの依存サービス | Aspire integration を優先 |
| Node.js / Python / Go など | 公式または Community Toolkit の polyglot API を検討 |
| 既存 Docker Compose から移行 | service を resource に置き換え、`depends_on` は `.WithReference()` / `.WaitFor()` に変換 |
| 起動順序が重要 | `.WaitFor()` と health check を使う |
| 外部 SaaS 接続 | parameter / secret / configuration で注入し、値を直書きしない |

---

## 7. ServiceDefaults と監視

各サービスには ServiceDefaults を適用し、ヘルスチェック、OpenTelemetry、サービス検出、標準的な resilience 設定を揃える。

確認するポイント:

- `AddServiceDefaults()` が各サービスで呼ばれているか
- `/health` や `/alive` などの health endpoint が定義されているか
- OpenTelemetry の traces / metrics / logs が Aspire dashboard で確認できるか
- 日本語ログ本文だけに依存せず、構造化ログのプロパティで検索できるか
- 個人情報や機密情報がログに出ていないか

ログメッセージは日本語でもよいが、検索・集計用のキーは英語にする。

```csharp
logger.LogInformation(
    "OrderCreated: {OrderId} {CustomerId}",
    orderId,
    customerId);
```

---

## 8. 設定とシークレット

設定値は次の優先順位で扱う。

1. Aspire integration / service discovery で自動注入できるものは直書きしない
2. ローカル秘密情報は User Secrets または環境変数
3. CI/CD は GitHub Actions secrets などの secret store
4. 本番は Azure Key Vault などのマネージド secret store

避けること:

- `appsettings.json` にパスワードや接続文字列を保存する
- AppHost に本番シークレットを直書きする
- 日本語の説明コメント内に実値のキーやトークンを貼る
- `.env` を無条件でコミットする

---

## 9. テスト方針

Aspire の変更では、単体テストだけでなく AppHost を含む統合テストを検討する。

| 変更内容 | 推奨テスト |
|---|---|
| AppHost の resource 追加 | AppHost 起動テスト、依存 resource の存在確認 |
| API と DB / Redis の連携 | 統合テスト |
| ServiceDefaults の変更 | health endpoint、OpenTelemetry 出力確認 |
| 設定変更 | Development / CI / Production 相当の設定読み込み確認 |
| デプロイ設定 | publish 生成物、環境変数、secret 参照確認 |

代表的な確認コマンド:

```powershell
dotnet restore
dotnet build
dotnet test
aspire run
```

CI では `aspire run` がそのまま使いにくい場合があるため、AppHost integration test または publish 検証に分ける。

---

## 10. デプロイ方針

デプロイ先を決めるときは、次を確認する。

- チームが運用できる基盤か
- Azure Japan リージョンで必要なサービスが利用可能か
- コスト、可用性、データ所在地要件を満たすか
- secrets / managed identity / network isolation をどう扱うか
- ログ、メトリクス、トレースの保管先と閲覧権限

Azure Container Apps を使う場合は、AppHost の resource と環境変数、managed identity、Key Vault 連携を明確にする。

---

## 11. トラブルシュート観点

| 症状 | 確認すること |
|---|---|
| `aspire run` が失敗する | .NET SDK、Aspire CLI、コンテナランタイム、ポート競合 |
| dashboard が開かない | AppHost の起動ログ、URL、ブラウザ、プロキシ |
| サービス間通信できない | `.WithReference()`、service discovery 名、HTTP/HTTPS endpoint |
| DB に接続できない | integration 名、接続文字列、container health、migration |
| 日本語が文字化けする | ファイル encoding、DB collation、console code page、HTTP charset |
| CI だけ失敗する | secret 設定、container runtime、SDK バージョン、タイムアウト |
| Azure デプロイ失敗 | リージョン対応、権限、quota、resource provider、managed identity |

Windows のコンソールで文字化けする場合は、PowerShell 7 と UTF-8 設定を優先して確認する。

---

## 12. レビュー時のチェックリスト

- [ ] AppHost の resource 名が安定していて、service discovery 名として妥当
- [ ] `.WithReference()` と `.WaitFor()` が依存関係に合っている
- [ ] シークレットや接続文字列が直書きされていない
- [ ] ServiceDefaults が適用され、health check と telemetry が機能する
- [ ] `dotnet restore` / `dotnet build` / `dotnet test` が通る
- [ ] `aspire run` で dashboard、ログ、traces、metrics を確認できる
- [ ] 日本語文字列、タイムゾーン、encoding の扱いが要件に合っている
- [ ] Azure を使う場合、Japan リージョンでの利用可否と権限が確認されている
- [ ] README や運用手順に必要な日本語説明が追加されている

---

## 13. よくある依頼への対応

### 新規 Aspire アプリを作る

1. SDK / CLI / container runtime を確認
2. 適切な template を選ぶ
3. AppHost、ServiceDefaults、主要サービスを作成
4. `dotnet build` と `aspire run` で確認
5. README に日本語の起動手順を追加

### 既存アプリに Aspire を追加する

1. 既存構成と起動順序を調査
2. `aspire init` または手動で AppHost / ServiceDefaults を追加
3. 外部依存を Aspire resource として表現
4. 接続文字列を service discovery / integration に寄せる
5. テストと README を更新

### Docker Compose から移行する

1. `docker-compose.yml` の service、port、volume、environment、depends_on を整理
2. 各 service を Aspire resource に変換
3. 永続化 volume と secret の扱いを明確化
4. `.WaitFor()` と health check を追加
5. `aspire run` と dashboard で起動順・ログを確認

---

## 14. 重要 URL

| リソース | URL |
|---|---|
| Aspire 公式ドキュメント | https://aspire.dev |
| Aspire GitHub | https://github.com/dotnet/aspire |
| Aspire samples | https://github.com/dotnet/aspire-samples |
| Aspire docs repo | https://github.com/microsoft/aspire.dev |
| Community Toolkit | https://github.com/CommunityToolkit/Aspire |
| Aspire dashboard image | `mcr.microsoft.com/dotnet/aspire-dashboard` |
