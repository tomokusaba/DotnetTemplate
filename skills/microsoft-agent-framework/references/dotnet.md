# Microsoft Agent Framework for .NET — 日本語環境向け

.NET / C# プロジェクトで Microsoft Agent Framework を使うときの参照。

## Authoritative sources

- Docs: https://learn.microsoft.com/agent-framework/
- Repository: https://github.com/microsoft/agent-framework/tree/main/dotnet
- Samples: https://github.com/microsoft/agent-framework/tree/main/dotnet/samples

## Package

新規プロジェクトでは最新 docs と NuGet version を確認してから追加する。

```powershell
dotnet add package Microsoft.Agents.AI
```

`Microsoft.Extensions.AI` の抽象、provider package、Azure SDK package が別途必要になる場合がある。

## 基本構成

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

ChatClientAgent agent = new(
    chatClient,
    instructions: "あなたは業務を支援するアシスタントです。");
```

`chatClient` は `IChatClient` 実装。Azure OpenAI、Microsoft Foundry、OpenAI、その他 provider から構成する。

## DI 方針

```csharp
builder.Services.AddSingleton<IChatClient>(sp =>
{
    // Provider-specific setup here.
    // Use DefaultAzureCredential when Azure authentication is appropriate.
    throw new NotImplementedException();
});

builder.Services.AddScoped<SupportAgentService>();
```

application 層へ provider 固有 SDK 型を漏らさない。domain / application service は `IChatClient` または独自 interface に依存させる。

```csharp
public interface ISupportAgent
{
    Task<SupportAnswer> AskAsync(
        SupportQuestion question,
        CancellationToken cancellationToken);
}
```

## Async / cancellation

- agent 実行は async / await を使う。
- `CancellationToken` を受け取り、下位処理へ渡す。
- `.Result` / `.Wait()` を使わない。
- timeout と retry は provider / HTTP / policy 層で設計する。

## Tool calling

Tool は通常の .NET method を AI から呼べるようにするため、入力検証・権限・監査が重要。

方針:

- tool は小さく分ける。
- destructive tool は approval を挟む。
- tool 引数は強く型付けする。
- tool 内で secret を返さない。
- DB 更新や GitHub 操作は authorization を確認する。

## TimeProvider

日時を返す tool や business logic では `DateTime.Now` / `DateTimeOffset.Now` を使わない。

```csharp
public sealed class DateTool(TimeProvider timeProvider)
{
    public DateTimeOffset GetUtcNow()
    {
        return timeProvider.GetUtcNow();
    }
}
```

DI:

```csharp
builder.Services.AddSingleton(TimeProvider.System);
```

## Testing with xUnit

単体テストは実 model 呼び出しに依存しない。

テストするもの:

- prompt / instructions の組み立て。
- tool の入力検証。
- tool の権限チェック。
- structured output の validation。
- workflow routing。
- cancellation / timeout。
- `FakeTimeProvider` による時刻依存。

```csharp
public sealed class SupportAgentServiceTests
{
    [Fact]
    public async Task AskAsync_InvalidQuestion_ThrowsArgumentException()
    {
        var service = new SupportAgentService(new FakeChatClient());

        await Assert.ThrowsAsync<ArgumentException>(
            () => service.AskAsync(new SupportQuestion(""), CancellationToken.None));
    }
}
```

## Observability

- `ILogger<T>` で structured logging。
- OpenTelemetry tracing を検討。
- model provider、deployment、latency、tool call count、error reason を記録。
- prompt / response 全文は機密情報を含む可能性があるため、既定でログに出さない。

## Review checklist

- [ ] `Microsoft.Agents.AI` と `Microsoft.Extensions.AI` の役割が明確
- [ ] provider 固有型が application 層へ漏れていない
- [ ] DI / Options / logging が .NET 標準に沿っている
- [ ] `CancellationToken` が伝播している
- [ ] tool に入力検証・権限・監査がある
- [ ] xUnit テストが実 model に依存しない
- [ ] `TimeProvider` を使っている
