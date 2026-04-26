---
name: dotnet-best-practices
description: ".NET / C# の設計、DI、設定、ログ、例外、非同期、xUnit テスト、TimeProvider、Microsoft Agent Framework による AI 操作を日本語環境向けに支援する Skill。Windows/PowerShell、UTF-8、Asia/Tokyo、ローカライズ、セキュリティ、CI 品質を扱う。"
license: MIT
---

# .NET / C# Best Practices — 日本語環境向け Skill

この Skill は、.NET / C# のコードを日本語チームで保守しやすく、安全でテストしやすい形に整えるために使う。設計、依存性注入、設定、ログ、例外処理、非同期処理、テスト、AI 操作、時刻依存処理を対象にする。

> **基本方針:** コードはテスト可能で、依存関係が明示され、環境差分に強いことを優先する。テストは xUnit を使う。AI 操作は Microsoft Agent Framework を使う。現在時刻は `DateTime.Now` や `DateTimeOffset.Now` ではなく `TimeProvider` から取得する。

---

## 既定アーキテクチャ

指定がない新規 .NET アプリケーションでは、最新安定版 ASP.NET Core、Blazor Web App の Interactive Server、Fluent UI Blazor、.NET Aspire AppHost / ServiceDefaults を既定にする。永続化が必要な場合だけ EF Core を使い、DB が必要な場合は SQL Server を第一候補にする。preview / RC、認証、過剰な layer は明示要件がある場合だけ追加する。

---

## 1. まず確認すること

| 確認項目 | 見る場所 / コマンド |
|---|---|
| .NET SDK | `global.json`, `dotnet --info` |
| Target Framework | `.csproj` の `TargetFramework` / `TargetFrameworks` |
| C# 言語バージョン | `.csproj` の `LangVersion`, SDK 既定値 |
| 共通ビルド設定 | `Directory.Build.props`, `Directory.Build.targets` |
| パッケージ管理 | `Directory.Packages.props`, Central Package Management |
| DI / Host 構成 | `Program.cs`, `HostApplicationBuilder`, `WebApplicationBuilder` |
| テスト方針 | `*.Tests.csproj`, xUnit パッケージ |
| 時刻依存処理 | `DateTime.Now`, `DateTimeOffset.Now`, `TimeProvider` |
| AI 操作 | `Microsoft.Agents.AI`, `Microsoft.Extensions.AI`, Agent Framework 設定 |
| CI | GitHub Actions, Azure Pipelines, `dotnet test` |

既存の `.editorconfig`、命名規則、プロジェクト構成、CI の警告設定がある場合はそれを優先する。

---

## 2. 日本語環境でのルール

- 説明、PR コメント、README、運用手順は日本語を基本にする。
- 型名、メソッド名、変数名、ログの構造化キー、設定キーは英語を基本にする。
- 日本語文字列を扱うファイルは UTF-8 を前提にする。
- Windows 手順では PowerShell と `\` 区切りのパスを優先する。
- 日時の保存・比較・ビジネスロジックは UTC を基本にし、表示時に `Asia/Tokyo` などへ変換する。
- ローカライズ対象の UI 文言やエラー文言はリソース化する。
- シークレット、接続文字列、API キー、プロンプト内の機密情報をリポジトリに含めない。

---

## 3. アーキテクチャと設計

推奨:

- 責務が明確な小さなクラスに分割する。
- ドメインロジック、インフラ、UI / API、AI 操作を分離する。
- 外部依存はインターフェイスや抽象化を通す。
- 複雑な生成処理は Factory / Builder を使う。
- コマンドやユースケースは Handler パターンを検討する。
- 公開 API には XML ドキュメントコメントを付ける。
- C# の新機能は、対象 SDK と CI が対応している場合に使う。

避けること:

- 1 クラスに複数責務を詰め込む。
- static 参照で時刻、設定、外部サービスに直接依存する。
- インフラ実装をドメイン層に漏らす。
- 例外や失敗を握りつぶす。
- テストのために本番コードの安全性を下げる。

---

## 4. Dependency Injection

依存関係はコンストラクター注入を基本にする。C# の primary constructor を使える場合は、既存スタイルと可読性を見て採用する。

```csharp
public sealed class OrderService(
    IOrderRepository repository,
    TimeProvider timeProvider,
    ILogger<OrderService> logger)
{
    public async Task<Order> CreateAsync(
        CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        var createdAt = timeProvider.GetUtcNow();
        logger.LogInformation(
            "Creating order for customer {CustomerId}",
            request.CustomerId);

        var order = Order.Create(request.CustomerId, createdAt);
        await repository.SaveAsync(order, cancellationToken);
        return order;
    }
}
```

登録例:

```csharp
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<OrderService>();
```

ライフタイムの目安:

| ライフタイム | 用途 |
|---|---|
| Singleton | stateless service、共有 cache、`TimeProvider.System` |
| Scoped | Web request 単位のサービス、DbContext 関連 |
| Transient | 軽量で状態を持たない短命サービス |

Singleton から Scoped を直接参照しない。

---

## 5. TimeProvider を必ず使う

現在時刻は `DateTime.Now`、`DateTime.UtcNow`、`DateTimeOffset.Now`、`DateTimeOffset.UtcNow` を直接使わない。`TimeProvider` を DI で注入し、`GetUtcNow()` を使う。

推奨:

```csharp
public sealed class SubscriptionService(TimeProvider timeProvider)
{
    public bool IsExpired(DateTimeOffset expiresAt)
    {
        return expiresAt <= timeProvider.GetUtcNow();
    }
}
```

避ける:

```csharp
public bool IsExpired(DateTimeOffset expiresAt)
{
    return expiresAt <= DateTimeOffset.Now;
}
```

### テストでの FakeTimeProvider

時刻依存テストでは `Microsoft.Extensions.TimeProvider.Testing` の `FakeTimeProvider` を使う。

```csharp
using Microsoft.Extensions.Time.Testing;

public sealed class SubscriptionServiceTests
{
    [Fact]
    public void IsExpired_WhenExpirationIsPast_ReturnsTrue()
    {
        // Arrange
        var now = new DateTimeOffset(2026, 4, 26, 0, 0, 0, TimeSpan.Zero);
        var timeProvider = new FakeTimeProvider(now);
        var service = new SubscriptionService(timeProvider);

        // Act
        var result = service.IsExpired(now.AddSeconds(-1));

        // Assert
        Assert.True(result);
    }
}
```

方針:

- ビジネスロジックは UTC で扱う。
- ローカル時刻が必要な場合だけ `GetLocalNow()` または `TimeZoneInfo` で変換する。
- テスト内で実時刻と fake time を混在させない。
- タイマーを使う場合は `TimeProvider.CreateTimer()` と破棄を意識する。
- 境界値として日付変更、月末、うるう年、夏時間、タイムゾーン差をテストする。

---

## 6. 非同期処理

I/O、外部 API、DB、ファイル、AI 呼び出しは async / await を使う。

推奨:

- 非同期メソッドは `Task` / `Task<T>` / `ValueTask<T>` を返す。
- `CancellationToken` を受け取り、下位呼び出しへ渡す。
- `.Result`、`.Wait()`、`GetAwaiter().GetResult()` は避ける。
- 例外は必要な粒度で捕捉し、文脈を追加して再送出する。
- ライブラリコードでは必要に応じて `ConfigureAwait(false)` を検討する。

```csharp
public async Task<User?> FindAsync(
    string userId,
    CancellationToken cancellationToken)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(userId);

    return await repository.FindAsync(userId, cancellationToken);
}
```

---

## 7. 設定と Options

設定は strongly-typed options と validation を使う。

```csharp
public sealed class PaymentOptions
{
    [Required]
    public required string Endpoint { get; init; }

    [Range(1, 300)]
    public int TimeoutSeconds { get; init; } = 30;
}
```

登録例:

```csharp
builder.Services
    .AddOptions<PaymentOptions>()
    .Bind(builder.Configuration.GetSection("Payment"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

方針:

- secret は `appsettings.json` に保存しない。
- ローカルは User Secrets または環境変数を使う。
- CI/CD は GitHub Actions secrets などを使う。
- 本番は Azure Key Vault などの secret store を使う。
- 設定不足は起動時に検出する。

---

## 8. ローカライズと日本語メッセージ

UI 文言、ユーザー向けエラー、通知文はリソース化する。ログの構造化キーは英語にし、メッセージ本文に日本語を使う場合も検索性を保つ。

推奨:

- ASP.NET Core では `IStringLocalizer<T>` を検討する。
- ライブラリでは `ResourceManager` を検討する。
- `ErrorMessages.resx`、`LogMessages.resx` など用途別に分ける。
- ユーザー向けメッセージと開発者向けログを混同しない。

```csharp
logger.LogWarning(
    "Order validation failed. OrderId: {OrderId}, Reason: {Reason}",
    orderId,
    reasonCode);
```

機密情報や個人情報をログに出さない。

---

## 9. 例外処理と入力検証

入力検証では標準の guard API を優先する。

```csharp
ArgumentNullException.ThrowIfNull(request);
ArgumentException.ThrowIfNullOrWhiteSpace(request.CustomerId);
```

方針:

- 具体的な例外型を使う。
- 例外を握りつぶさない。
- 想定可能な業務エラーは Result 型やエラーコードも検討する。
- catch する場合は、復旧・変換・ログ付けなど明確な目的を持つ。
- ユーザー向けメッセージとログ詳細を分ける。

避ける:

```csharp
try
{
    await DoWorkAsync();
}
catch
{
    return;
}
```

---

## 10. ログと OpenTelemetry

ログは `Microsoft.Extensions.Logging` を使い、構造化ログにする。

```csharp
logger.LogInformation(
    "Payment completed. PaymentId: {PaymentId}, Amount: {Amount}",
    paymentId,
    amount);
```

方針:

- ログテンプレートのキーは英語にする。
- 例外ログでは `logger.LogError(exception, "...")` を使う。
- 相関 ID、ユーザー ID、テナント ID などは必要最小限にする。
- 分散アプリでは OpenTelemetry の traces / metrics / logs を検討する。
- AI 操作ではプロンプト、応答、ツール呼び出しの記録範囲をセキュリティ方針に合わせる。

---

## 11. テストは xUnit を使う

テストフレームワークは xUnit を使う。MSTest や NUnit は既存プロジェクトが採用済みでない限り新規採用しない。

基本パッケージ:

```xml
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="..." />
<PackageReference Include="xunit" Version="..." />
<PackageReference Include="xunit.runner.visualstudio" Version="..." />
```

時刻依存テスト:

```xml
<PackageReference Include="Microsoft.Extensions.TimeProvider.Testing" Version="..." />
```

方針:

- Arrange-Act-Assert を使う。
- テスト名は `MethodName_Scenario_ExpectedBehavior` を基本にする。
- 1 テストは 1 つの振る舞いを検証する。
- 成功・失敗・境界値をテストする。
- 外部 API、DB、時刻、乱数、ファイルシステムは fake / mock / test double で制御する。
- async テストは `async Task` にする。
- `.Result` や `.Wait()` を使わない。
- ロケール依存処理は `CultureInfo("ja-JP")` または `CultureInfo.InvariantCulture` を明示する。

例:

```csharp
public sealed class OrderServiceTests
{
    [Fact]
    public async Task CreateAsync_ValidRequest_SavesOrder()
    {
        // Arrange
        var timeProvider = new FakeTimeProvider(
            new DateTimeOffset(2026, 4, 26, 0, 0, 0, TimeSpan.Zero));
        var repository = new InMemoryOrderRepository();
        var service = new OrderService(repository, timeProvider, NullLogger<OrderService>.Instance);

        var request = new CreateOrderRequest("customer-001");

        // Act
        var order = await service.CreateAsync(request, CancellationToken.None);

        // Assert
        Assert.Equal("customer-001", order.CustomerId);
        Assert.Equal(timeProvider.GetUtcNow(), order.CreatedAt);
    }
}
```

---

## 12. Microsoft Agent Framework による AI 操作

AI 操作は Microsoft Agent Framework を使う。新規実装で Semantic Kernel を直接採用しない。既存コードが Semantic Kernel の場合は、Microsoft Agent Framework への移行または共存方針を検討する。

基本方針:

- Agent Framework の名前空間は `Microsoft.Agents.AI` を使う。
- メッセージや AI 抽象化には `Microsoft.Extensions.AI` を使う。
- シンプルな agent は `ChatClientAgent` と `IChatClient` で構成する。
- 複雑な処理は orchestration、workflow、tool calling、structured output を使う。
- provider は Azure OpenAI、Microsoft Foundry、OpenAI などから要件に合わせて選ぶ。
- secret は環境変数、User Secrets、Key Vault などから取得する。
- プロンプトインジェクション、データ漏えい、ツール実行権限を設計時に考慮する。
- OpenTelemetry による observability を検討する。

最小構成の考え方:

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

var agent = new ChatClientAgent(
    chatClient,
    instructions: "あなたは社内業務を支援するアシスタントです。");
```

設計ルール:

| 項目 | 方針 |
|---|---|
| 入力 | 信頼境界を明確にし、ユーザー入力を検証する |
| プロンプト | version 管理し、目的・制約・禁止事項を明示する |
| ツール | 最小権限、明示的な引数、監査可能な実装にする |
| 出力 | 可能なら structured output で受け取る |
| 記録 | 機密情報を除外し、必要な監査ログだけ残す |
| テスト | agent の外側に薄い adapter を置き、IChatClient や agent runner を差し替える |
| 運用 | rate limit、timeout、retry、circuit breaker を検討する |

避けること:

- API キーをソースコードやプロンプトに埋め込む。
- AI 応答を検証せずに DB 更新や外部送信へ使う。
- ツールに広すぎる権限を与える。
- ユーザー入力を system prompt と同じ信頼度で扱う。
- AI 呼び出しを単体テストで実モデルに依存させる。

---

## 13. データアクセスとセキュリティ

方針:

- SQL はパラメーター化する。
- EF Core では必要に応じて projection、pagination、tracking 無効化を使う。
- 個人情報、認証情報、トークンをログに出さない。
- 入力値は境界で検証する。
- 認可は UI だけでなく API / service 層でも確認する。
- 暗号化や署名は標準ライブラリと管理された key store を使う。
- 外部 API 呼び出しには timeout、retry、cancellation を設定する。

---

## 14. パフォーマンス

推奨:

- 不要な allocation を避ける。
- hot path では `Span<T>` / `ReadOnlySpan<T>` を検討する。
- I/O は非同期化する。
- 大量データは streaming / pagination を使う。
- `HttpClient` は `IHttpClientFactory` を使う。
- キャッシュは有効期限、無効化、メモリ上限を設計する。
- LINQ の多重列挙に注意する。

ただし、可読性を大きく犠牲にする最適化は、測定結果がある場合だけ行う。

---

## 15. コード品質

チェック項目:

- SOLID 原則に反していない。
- メソッドが大きすぎない。
- ドメイン用語が名前に反映されている。
- nullability warning を放置していない。
- analyzer warning を理解せず抑制していない。
- public API には XML docs がある。
- resource disposal が正しい。
- `CancellationToken` が下位処理に伝播している。
- `DateTime.Now` / `DateTimeOffset.Now` を使っていない。
- AI 操作が Microsoft Agent Framework に集約されている。

---

## 16. 公式参照

| リソース | URL |
|---|---|
| .NET documentation | https://learn.microsoft.com/ja-jp/dotnet/ |
| C# documentation | https://learn.microsoft.com/ja-jp/dotnet/csharp/ |
| TimeProvider overview | https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview |
| Testing with FakeTimeProvider | https://learn.microsoft.com/dotnet/core/extensions/timeprovider-testing |
| Microsoft Agent Framework | https://learn.microsoft.com/agent-framework/overview/agent-framework-overview |
| Microsoft.Extensions.AI | https://learn.microsoft.com/dotnet/ai/microsoft-extensions-ai |
| xUnit | https://xunit.net/ |
